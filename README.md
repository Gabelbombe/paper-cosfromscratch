# Defining and Building a Container Orchestration System

In this paper I will be explaining why HashiCorp Nomad is a chosen core component of a cluster orchestration system, how the architecture of such a system could look like and discuss the components covering some of the most important topics like job scheduling, service discovery and load balancing/routing of ingress traffic.

Later I will show you how to set up the Container Orchestration System as it is I have outlined. The whole setup will be made using Terraform on an empty AWS account to ensure that it can be automated, reproduced and thus easily maintainable. The goal is to have a complete, [soup to nuts](https://en.wikipedia.org/wiki/Soup_to_nuts) platform where services can be deployed and managed easily.

![Visualizing a CoS](assets/container-gateway.jpg)


## What a Container Orchestration System could look like

If you jumped on the containerization train, and dockerized your application components (ala, microservices) you are on a good path for a scalable and resilient system.

To really run such a system though, at production at scale the questions to be answered are:

 1. **Scheduling:** Where do these containers run?
 2. **Management:** Who manages their life cycle?
 3. **Service Discovery:** How do they find each other?
 4. **Load Balancing:** How to route requests?

After some research one quickly finds systems like [Kubernetes](https://kubernetes.io/), [DC/OS](https://dcos.io/), [AWS Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html), [AWS EKS](https://aws.amazon.com/eks/) (An AWS managed kubernetes cluster), [Docker Swarm](https://github.com/docker/swarm/) and of this writing, probably a million more. Such systems are _kind of_ orchestrating the containers placement, communication and life cycle. The so called **Container Orchestration Systems** are responsible to manage containers and to abstract away the actual location they are running on.

All of these technologies have their advantages, disadvantages and of course a different feature sets. For example, DC/OS has really a big feature set, but it is very hard due to the learning curve to set up a production ready DC/OS cluster. Using kubernetes on Google Cloud is a good idea, but as soon as you want to spin it up on an AWS environment you will have a bad time.

These problems have vanished with AWS EKS, but since this service is relatively new there are important features that are missing. Additionally, and even more importantly, with AWS EKS you lose the option to run a hybrid multi-IaaS provider platform. With the abstraction you've gained by using containers there is lot of potential in offloading components on cheaper platforms like Microsoft Azure or even regional data centers. Thus looking at the costs it is a good idea to keep this option.


### Nomad as Core Component

After looking at the mentioned Container Orchestration Systems there is one I would like to focus on, this is other product called [Nomad](https://www.nomadproject.io/). **Nomad is a scheduler of applications and services no more, no less.** Nomad can't, or should it compete with the feature sets provided by kubernetes or DC/OS. All the important features for managing and running services are available. It does _just_ one job but does it extremely well.

Other features like service discovery, load balancing, secret management, monitoring and logging are available open source and can be folded in easily to expand its capabilities.

Nomad is developed by [Hashicorp](https://www.hashicorp.com/), a company who's mantra is _Consistent workflows to provision, secure, connect, and run any infrastructure for any application._.


Having the bigger picture in mind, the Hashicorp engineers know exactly what the important components are, and how to implement these tools in a robust suite. Besides Nomad, they provide [Consul](https://www.consul.io/) (A Service Discovery and Connectivity tool), [Vault](https://www.hashicorp.com/products/vault/) (A Secrets as a Service tool) and [Terraform]() (An Infrastructure provisioning tool). All of them integrate with Nomad, layering in the missing features, such as service discovery to the Container Orchestration System to be set up.
