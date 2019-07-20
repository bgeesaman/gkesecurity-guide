# The Unofficial Google Kubernetes Engine (GKE) Security Guide

## Introduction

!!! warning "Disclaimer"

    This guide is not an official guide, is not endorsed by [Google](https://google.com), and is comprised entirely from publicly available information.  The information is provided "as-is", without warranty or fit for a particular purpose.  You should fully understand the security implications of each recommendation before applying it in your environment.

There are several books on the topic of getting [Kubernetes Up and Running](http://shop.oreilly.com/product/0636920043874.do) and even one specific to [Kubernetes Security](https://kubernetes-security.info/). If you are deploying in multiple platform environments like on-premise or across multiple cloud providers, those resources will be very useful.  However, if you are deploying GKE clusters in GCP, you'll notice that there are a wide array of GCP and GKE-specific choices that need to be made.  The goal of this guide is to help you prioritize and implement a security posture that meets your organization's needs while taking advantage of all the benefits of the GKE service.

For each control and approach, the guide will attempt to help you understand the relative risks, relative likelihood, upfront cost and complexity, and ongoing cost and complexity of each security-related decision.  Deep links to official GCP/GKE and Kubernetes documentation will be provided to guide further reading.  Finally, links to hand-selected tools, books, and talk recordings will give additional support to the concepts for further exploration and understanding.

## Audience

This guide is focused specifically on the following roles working with a GKE deployment:

* Operations - Those responsible for deploying and maintaining the GCP Project, Networking, and GKE Cluster lifecycle.
* Security - Those responsible for ensuring the operational environment adequately meets the organization's security standards and posture.
* Security-minded developers - Those deploying operational workloads into the GKE cluster looking to follow security best practices.
* Team Leads - Those managing teams implementing GKE clusters and responsible for prioritizing security with operational velocity.

## Structure of this Guide

As each layer of a system builds upon the layer underneath, this guide begins with the foundation of all GCP services and explains how the structure of folders and projects in a GCP organization can help organize and guide security settings and permissions management.  Next, it covers the types of network configurations that support GKE clusters for different isolation strategies.  The core of the guide gets into cluster configuration, lifecycle and operations, observability, and the management of custom additions in the form of addons.  With a solid base in place, the guide covers the deployment and configuration of workloads and container images in terms of security best practices.  Finally, it covers certain security-related workflows such as auditing for conformance and handling security issues when an incident occurs.

Each topic area will cover:

* A few sentences to paragraphs on what the topic or setting is and does.
* Why the topic or setting is important and the risks associated with it.
* The versions it applies to, where applicable.
* When you might want to prioritize adoption or implementation.
* The upfront cost and complexity of deploying the change or approach.
* The longer term cost and complexity of maintaining the system with this change or approach implemented.

## GKE vs Building Kubernetes Yourself

For first time Kubernetes operators, it's a fantastic idea to follow [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) by Kelsey Hightower and build a cluster from scratch.  It will give you a deep understanding of the components, their configuration settings, and an appreciation for the complexity of the system.  The very next thing you should do is leverage an approach to Kubernetes that ensures you and your team do as little of that work as possible.

In many cases, this is best done by working with a managed Kubernetes service like [GKE](https://cloud.google.com/kubernetes-engine/).  Successful Kubernetes cluster deployments require solid execution in wide range of areas, and it's impossible to be execellent in all of them at once.  From a business standpoint, you want your teams working on the problems that matter to your organization and to leave the boring operational work to the cloud service provider.  Kubernetes version upgrades, security patches, operating system maintenance, logging and monitoring infrastructure, and more are all things you want to be building "on top of" and not having to "build" yourself first.  Refer to the [benefits of GKE](TODO) for further reading.

## Kubernetes Security Maturity

Throughout this guide, there will be dozens of security decisions and configuration tasks that you will be faced with prioritizing and implementing.  It's important to have some higher level structure and frame of reference for which items are required early or later on in your journey to Kubernetes security maturity.  [Kubernetes for Enterprise Security Requirements](https://www.youtube.com/watch?v=X-rSMkKqyt4) really helps visually explain the progression path well.

Summarizing the key areas from this talk:

* Infrastructure Security - Does the surrounding environment and Kubernetes cluster provide a solid foundation for running workloads securely?
* Software Supply Chain - Are the container images running in the cluster built from trusted sources and free of vulnerabilities?
* Container Runtime Security - Are there capabilities in place to monitor what is happening inside the containers?

Additionally:

* Security Observability and Response - Are there capabilities for knowing when a security incident occurs and processes in place to respond and remediate?

As you progress through this guide, each of the topic areas should map back to one or more of these key areas and help align your decision along these lines.

## Contributing

This guide is a living document, and contributions in the form of issues and PRs are welcomed.  If you are considering writing new content, please open an issue outlining what you'd like to write about, where it might fit in, and other details.

If you found this guide useful, please consider donating your time attending and supporting your local cloud-native and Kubernetes-related meetups.  The success or failure of Kubernetes and the CNCF ecosystem is largely dependent on you and how you help elevate others with your compassion, assistance, and inclusivity.

## About the Author(s)

* [@BradGeesaman](https://twitter.com/bradgeesaman) is an Independent Security Consultant currently focused on helping teams secure their GKE clusters in harmony with their desired operational velocity.
