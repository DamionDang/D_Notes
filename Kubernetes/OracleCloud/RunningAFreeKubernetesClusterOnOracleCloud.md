# Running A Free Kubernetes Cluster On Oracle Cloud

Free? Kubernetes? On Oracle Cloud? What?

Yeah, that’s right. You can totally run a completely free managed Kubernetes cluster on Oracle Cloud. They have a very genereous offer none of the other mainstream cloud providers have (AWS/GCP).

This article is going to be only the beginning of this series. I’ll show you what you can build totally free on Oracle Cloud and I’ll also show the how-to using Terraform. And we’re not gonna stop there. I’ll also share how I set up a CI/CD pipeline – though a very simple one – for this free cluster using GitHub Actions.

Let’s jump into the topic.

## The Requirement

Let’s see what’s the minimal infrastructure requirement for a cluster like this on the cloud. I’m going to use the AWS terminology just for the sake of not looking up each cloud’s own one.

- A VPC
- A private subnet for the cluster nodes
- A public subnet where you can put the load balancer, NAT gateway, etc.
- A NAT gateway for the private nodes to download updates/etc
- An internet gateway to route traffic from/to the internet
- EC2 instances as cluster nodes
- Preferably multiple Availability Zones
- A Kubernetes cluster
- A private Docker registry
- A load balancer to route traffic into the cluster from the outside

A deployment view could look the following:![img](https://arnoldgalovics.com/wp-content/uploads/2022/02/Blank-diagram-2-1024x701.png)

Since my initial intent was to create the cluster with a low cost, the most obvious approach was to use ARM based compute nodes and turns out Oracle has a very good offering on that front.

Let’s compare the costs with different cloud providers.

## Comparing Costs

Starting with AWS (for region eu-central-1):

- The VPC is free
- Private subnet is free
- Public subnet is free
- NAT gateway is not free. It has a flat hourly fee and 1 NAT gateway would cost **38.01** USD per month
- Internet gateway is free
- EC2 instances. Since I want to run the cluster in multi-AZ I chose t4g.large instances, 3 of them. It has 2 vCPUs and 8GB of RAM each. It’s an ARM instance. Monthly it costs **178.9 USD** with a 30GB gp2 SSD
- Kubernetes cluster on AWS has a flat hourly rate. 1 cluster costs **73 USD** per month
- Docker registry using ECR with 5 GB inbound & outbound traffic and 1 GB of storage per month. It’s essentially a calculation to store at most 1 docker image and 5 builds & deployments per month. Costs **0.2 USD** per month
- Elastic Load Balancing also has an hourly flat rate. I’m going with a single ALB with minimal traffic on it. Costs **19.72 USD** per month

That adds up to **309.83 USD** per month or **3 717.96 USD** per year.

GCP (region europe-west3), I’ll only cover the non-free things:

- NAT gateway with 1 GB of data processed: **1.07 USD** per month
- Compute instances. I chose 3 x e2-standard-2 with 2 vCPU and 8GB RAM although this is a non ARM instance. Turns out GCP doesn’t have ARM instances yet. Costs **189.07** per month.
- Docker registry costs **0.05 USD** per month with 1 GB storage per month
- Load Balancing costs **21.91 USD** per month with 1 forwarding rule and 1 GiB of traffic.
- Just a last note, the first zonal Kubernetes cluster is free.

That adds up to **212.10 USD** per month.

Oracle Cloud (region eu-frankfurt-1), I’ll also cover the non-free things:

- –

That’s right. All of this is free with Oracle Cloud. Just saved you 3 000 USD per year. More here on the [Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm). And I know the AWS/GCP setup has 2 more vCPUs than what the free Oracle Cloud offers but that was the closest setup I could get.

## Oracle Cloud Tooling For Kubernetes

First of all, I have to say if you’ve worked with AWS or GCP, the Oracle Cloud console interface is a big step-back. But I mean if you can save that amount of money, who cares right?

I personally struggled with the Oracle Cloud Console UI, especially around setting up MFA for my users but it’s okay-ish after a couple days. You just need to get used to it.

So, let’s see what are the service offerings from Oracle Cloud for hosting a Kubernetes cluster.

The VPC is called VCN here, Virtual Cloud Network. That’s where you can create your subnets, route tables, NAT gateways, Internet Gateways as well as manage your security lists. The VCN services are always free although there’s a cap on the the traffic going in or out. I can’t quite recall which one but I think the cap is like 10 TB.

The EC2 is simply called compute instances. Now the good thing. You get 2 AMD micro compute instances and you get at most 4 ARM based compute instances, each with 1 vCPU and 6 GB RAM. Overall **4 ARM vCPUs and 24 GB RAM**. Yeah, that’s right. They’re always free.

The Kubernetes cluster is called Container Engine for Kubernetes (OKE). I love the approach they’re taking. It’s completely free. Here’s a quote from their pricing page:

> There is no additional charge for cluster management using Container Engine for Kubernetes. Customers pay only for the infrastructure used by containerized workloads, such as for the worker nodes (compute), storage, and other resources consumed. Oracle manages the multi-availability domain parent nodes and provides them to customers for free.

I love it. I mean it was always strange for me that cloud providers are charging money for a Kubernetes cluster plus the underlying compute infrastructure. Especially the amount they do charge. Crazy.

The Docker registry is called Container Registry. There’s no additional costs to this service except the storage of the image which they’re charging on the same rate as their Object Storage (S3 in Oracle Cloud). But, the always free tier includes 20 GB of storage which is way more than enough.

Load balancing. There are 2 types of load balancers here just like for AWS. The ALB is called Flexible Load Balancer. The Network Load Balancer is called just the same. One Flexible Load Balancer is always free with a bandwidth cap of 10 Mbps. A single Network Load Balancer is always free and this is where it’s a little bit strange for me as well. Their pricing page says that an [unlimited amount of NLBs are also free](https://www.oracle.com/cloud/networking/load-balancing-pricing.html). Probably just a typo, I’m not sure but for this setup we’re more than happy with a single load balancer of any type.

## Summary

This post is a little short but there’s more to come. The impotant thing here is if you want to set up a Kubernetes cluster with zero cost; or at least you don’t want to pay $200+ per month for a Kubernetes cluster. Oracle Cloud can help you.

Bear in mind though since the nodes are ARM instances, there’s some magic involved to build the docker images to be compatible with the ARM architecture.

Next up, I’ll show you how I set up a fully functioning Kubernetes cluster using Terraform on Oracle Cloud. Then we’ll check how you can build docker images for an ARM cluster; or as a matter of fact for a hybrid cluster too.

>Some articles, data, and pictures come from the Internet, and all copyrights belong to the original website or the original author.
>
>If your rights are violated, please write to inform us to delete it.