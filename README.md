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

Nomad is developed by [Hashicorp](https://www.hashicorp.com/), a company who's mantra is _"Consistent workflows to provision, secure, connect, and run any infrastructure for any application."_.


Having the bigger picture in mind, the Hashicorp engineers know exactly what the important components are, and how to implement these tools in a robust suite. Besides Nomad, they provide [Consul](https://www.consul.io/) (A Service Discovery and Connectivity tool), [Vault](https://www.hashicorp.com/products/vault/) (A Secrets as a Service tool) and [Terraform]() (An Infrastructure provisioning tool). All of them integrate with Nomad, layering in the missing features, such as service discovery to the Container Orchestration System to be set up.

To summarize - the most useful features that brought me to Nomad are:

 - **Complexity:** Nomad is easy to understand, therefore to set up and maintain.
 - **Job Control:** Nomad is a highly scalable scheduler using an _evaluation_ approach.
 - **Container/Task Support:** Docker, Rkt, simple binaries/executables and even bash scripts can be scheduled.
 - **Cloud Provider Agnostic:** Hybrid as well as multi-IaaS provider setup is possible.
 - **Extensibility:** Well thought out integration with other Hashicorp tools, to implement missing feature sets.
 - **Deployment:** Supports known deployment design patterns, such as rolling, canary and blue green.


### Architectural Overview

![Architectural Component Overview](assets/arch-overview.png)

The core of our system  then will be Nomad. Nomad is able to deploy and manage services/applications on a fleet of compute instances (client nodes).

Nomad manages these in from a templating systems called [Nomad jobs](https://www.nomadproject.io/docs/job-specification/index.html). A **Nomad job** is either a single task or a group of tasks which, as said, can be a container (docker or rocket), a raw binary executable, a jar file or even a bash script. Such a job represents all the things that have to be deployed tightly together on the same client node.

Nomad itself is shipped as a simple binary that provides a server and a client mode. This binary is deployed on some type of compute instances (i.e AWS EC2) thus transforming these instances to Nomad server or client nodes.

At least three instances (a cluster quorum) with the Nomad binary in **server mode** per data center are used. This implementation leasds to a fault tolerant Nomad cluster across multiple availability zones. In the image above they are marked with the green Nomad logos. Using the [raft protocol](https://raft.github.io/) the server nodes elect a Nomad leader. The leader is then responsible management of all cluster calls and decisions.

Instances with the nomad binary in **client mode**, will be the nodes where the actual jobs are deployed and ran. In the image above these nodes are indicated by the smaller boxes.

Nomad also provides a feature called federation, which enables the option of connecting different Nomad clusters. Having this implemented the system can orchestrate and manage services across multiple data centers, which even can be hosted by different cloud providers. Indicated by the bold purple line in the architecural diagram, the Nomad leader of _Data-Center A (aka, us-east-1)_ communicates with the leader in _Data-Center B (aka, us-east-2)_ using [Serf](https://www.serf.io/) (A lightweight gossip protocol).


### Service Discovery

![Service Discovery Mesh](assets/service-discovery.png)

Beside Nomad, Consul will also be an essential part of our system, which gives the answer to two important questions:

 1. How do the Nomad server nodes find each other, and how do they know about the state and location of the client nodes?
 2. How can services find other services they will be need to communicate with?

The issue of the lack of service discovery is solved by implementing Consul. Consul knows the current health status and the location (IP and port, etc) of all registered services.

Just like Nomad, Consul is a single binary that can be ran in either server or client mode. The Consul agent in **server mode** is deployed to instances which are then transformed into the server nodes. In the architectural overview image, these nodes are marked with the purple consul icon. For fault tolerance at least three consul server nodes are deployed. They elect (like nomad does) a leader, that manages all cluster calls and decisions.

**Consul in client mode** runs on the remaining instances, which are the nomad server and nomad client nodes. Instead of contacting the consul server, each component directly communicates with the client that is locally available on each node. This removes the need to find out the actual location of the consul server.

Nomad and Consul are perfectly integrated, therefore the nomad nodes are able to find each other automatically using consul. Each node, either server or client, registers itself with consul then reports its health status. The nomad nodes are then promoted through the consul API as available nomad client or nomad service.

The same feature is applied for the jobs management by nomad. Nomad automatically registers a deployed job at consul. Thus all deployed services can be found by querying the consul API, which then returns the concrete IP, port and health status of the specific service.

This implies that each service has to implement code that queries the consul API. To avoid this effort there are components that can be used instead, like the load balancer [fabio](https://fabiolb.net/) or [envoy](https://www.envoyproxy.io/) which creates a service mesh.

To ease up the first setup, fabio will be used. Envoy will be introduced instead in an upcoming paper as it is a better solution for our purpose.


### Ingress Controller

![Ingress Traffic Controller](assets/ingress-controller.png)

As mentioned, fabio will be used for now as our load balancer and ingress traffic controller. Fabio integrates well with consul by implementing and leveraging consuls native API. Internally fabio has knowledge of consuls service catalog and thus about the state and location of the services registered with consul. Based on this knowledge fabio adjusts IP rules and routing tables on the specific nomad client nodes, this enables requests to be routed to the correct targets. It even works if the requested job lives on another instance, since the routes are based on IP and port.

This scenario is illustrated in the image above. Here the client requests a service represented by job A on nomad. After hitting the AWS ALB the request is routed to fabio, deployed as nomad job, which then forwards the request to job A. Either to the instance of job A on nomad client node 1 or 2.


## Setting Up our Container Orchestration System

Now that we have outlined the technologies, lets roll out the big guns and start assembling the pieces to put our Container Orchestration System into action. As a note, all steps I will outline and scripts used here have been tested with an Ubuntu 16.04, but should also work on other linux based systems with minor tweaks and massages.

![Our Container Orchestration System](assets/cos-outline.png)


### Prerequisites

Before you can start with the rollout of the COS you need an AWS account, as well as the installation tools mentioned below.


#### AWS Account and Credentials Profile

Feel free to skip this account setup section if you already own or manage an AWS account.

The COS code is written in Terraform using the AWS services provider. You will need an AWS account to be able to deploy this system. To create a new account, just have to follow the tutorial: [Create and Activate an AWS Account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/).

Once you have an account, you will need to create AWS access keys using the AWS Web console:

 1. Login into your new AWS account.
 2. Create a new user for your account, by following this [tutorial](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html).
 3. Create a new access key for this user, by following this [tutorial](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey).

Once the above steps are complete, don’t forget to actually download your keys. This is the only time you can do this, so if you loose your keys you have to create a new keys following the same steps.

![Download your credentials](assets/aws-credentials.png)

The downloaded file `accessKeys.csv` contains the `Access key ID` and the `Secret access key`.

Now that you have your keys, you can create an AWS profile. A profile in this context is just a name referencing access keys and some options for the AWS account in use. In order to use these, you'll have to create or modify the file `~/.aws/credentials`.

In this file, just add a new section named `my_cos_account` pasting in the `Access key ID`, the `Secret access key` and save the file.

```ini
[my_cos_account]
aws_access_key_id = PASTE HERE YOUR ACCESS KEY
aws_secret_access_key = PASTE HERE YOUR SECRET KEY
```

Now with your `my_cos_account` profile, you can use it to directly create the AWS resources that are needed to build up the COS.


#### Tools

Before we can really start to deploy the COS we'll have to install some essential tools.

**Terraform:** Is needed to create AWS resources. Here version **0.11.11** was used.

 - Download the binary from [Terraform Downloads](https://www.terraform.io/downloads.html).
 - Unzip and install it.

```bash
cd ~/Downloads
unzip terraform_0.11.11_linux_amd64.zip
sudo mkdir -p /opt/terraform/0.11.11
sudo mv terraform /opt/terraform/0.11.11

cd /usr/bin
sudo ln -s /opt/terraform/0.11.11/terraform terraform
```

 - Test it with `terraform --version`

**Nomad (CLI):** Is also needed to be able to deploy services into the COS and show the status of the COS. Here version **0.8.6** was used.

 - Download the binary from [Nomad Downloads](https://www.nomadproject.io/downloads.html)
 - Unzip and install it.

```bash
cd ~/Downloads
unzip nomad_0.8.6_linux_amd64.zip
sudo mkdir -p /opt/nomad/0.8.6
sudo mv nomad /opt/nomad/0.8.6

cd /usr/bin
sudo ln -s /opt/nomad/0.8.6/nomad nomad
```

 - Test it with `nomad --version`

**Packer:** Is needed to bake (create) the AWS AMI that contains the nomad binary, which is then actually used as image for the AWS EC2 instances that form the COS. Here version **1.3.3** was used.

 - Download the binary from [Packer Downloads](https://www.packer.io/downloads.html)
 - Unzip and install it.

```bash
cd ~/Downloads
unzip packer_1.3.3_linux_amd64.zip
sudo mkdir -p /opt/packer/1.3.3
sudo mv packer /opt/packer/1.3.3

cd /usr/bin
sudo ln -s /opt/packer/1.3.3/packer packer
```

 - Test it with `packer --version`


### Deployment

The whole setup consists of terraform code and is available at [Terraform: A Container Orchestration System](https://github.com/ehime/terraform-cos).
This project is designed as a Terraform module with a tailored API. It can be directly integrated into an existing infrastructure adding in a Container Orchestration System.

Additionally this project provides a self contained `root-example`, that deploys beside the COS with minimal networking infrastructure. This example will be used here to roll out the system.

The following steps will be required:

 1. Obtain the source code from github.
 2. Build the Machine Image (AMI) for the EC2 instances.
 3. Create an EC2 instance key pair.
 4. Deploy the infrastructure and the COS.
 5. Deploy fabio.
 6. Deploy a sample service.
