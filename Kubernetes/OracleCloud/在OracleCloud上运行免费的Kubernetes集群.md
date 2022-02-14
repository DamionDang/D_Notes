# 在 Oracle Cloud 上运行免费的 Kubernetes 集群

自由？Kubernetes？在甲骨文云上？什么？

是的，没错。您可以在 Oracle Cloud 上完全运行一个完全免费的托管 Kubernetes 集群。他们提供了其他主流云提供商所没有的非常慷慨的报价（AWS/GCP）。

这篇文章只是这个系列的开始。我将向您展示可以在 Oracle Cloud 上完全免费构建的内容，还将展示如何使用 Terraform。我们不会止步于此。我还将分享我是如何使用 GitHub Actions 为这个免费集群设置 CI/CD 管道的——尽管是一个非常简单的管道。

让我们进入主题。

## 要求

让我们看看像这样的集群在云上的最低基础架构要求是什么。我将使用 AWS 术语只是为了不查找每个云自己的术语。

- 一个 VPC
- 集群节点的私有子网
- 一个公共子网，您可以在其中放置负载均衡器、NAT 网关等。
- 用于私有节点下载更新/等的 NAT 网关
- 一个互联网网关，用于路由来自/到互联网的流量
- EC2 实例作为集群节点
- 最好是多个可用区
- 一个 Kubernetes 集群
- 私有 Docker 注册表
- 负载均衡器，用于将流量从外部路由到集群

部署视图可能如下所示：![img](https://github.com/DamionDang/D_Notes/blob/d74d11a872a526eb29e1885bb47470142ea255be/Kubernetes/OracleCloud/image/image01.png)

由于我最初的意图是以低成本创建集群，因此最明显的方法是使用基于 ARM 的计算节点，结果证明 Oracle 在这方面提供了非常好的产品。

让我们比较不同云提供商的成本。

## 比较成本

从 AWS 开始（对于区域 eu-central-1）：

- VPC 是免费的
- 私有子网是免费的
- 公共子网是免费的
- NAT 网关不是免费的。它按小时收费，1 个 NAT 网关每月花费**38.01美元**
- 互联网网关是免费的
- EC2 实例。因为我想在多可用区中运行集群，所以我选择了 t4g.large 实例，其中 3 个。它有 2 个 vCPU 和 8GB 的 RAM。这是一个 ARM 实例。每月花费**178.9 美元**，配备 30GB gp2 SSD
- AWS 上的 Kubernetes 集群具有固定的小时费率。1 个集群的费用为每月**73 美元**
- 使用 ECR 的 Docker 注册表，每月有 5 GB 入站和出站流量和 1 GB 存储空间。它本质上是每月最多存储 1 个 docker 映像和 5 个构建和部署的计算。每月费用**0.2 美元**
- Elastic Load Balancing 还具有每小时固定费率。我将使用一个流量最小的 ALB。每月费用**19.72 美元**

每月总计 309.83 美元或每年**3** **717.96 美元**。

GCP (region europe-west3)，我只会介绍非免费的东西：

- 处理 1 GB 数据的 NAT 网关：每月**1.07 美元**
- 计算实例。我选择了 3 x e2-standard-2 和 2 个 vCPU 和 8GB RAM，尽管这是一个非 ARM 实例。原来 GCP 还没有 ARM 实例。每月**花费 189.07**。
- Docker 注册表每月花费**0.05 美元**，每月存储 1 GB
- 负载平衡每月花费**21.91 美元，包含 1 条转发规则和 1 GiB 流量。**
- 最后一点，第一个区域 Kubernetes 集群是免费的。

每月总计212.10**美元。**

Oracle Cloud（区域 eu-frankfurt-1），我还将介绍非免费的东西：

- –

那就对了。Oracle Cloud 提供的所有这些功能都是免费的。每年只需为您节省 3 000 美元。更多关于[永远免费资源](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)的信息。而且我知道 AWS/GCP 设置比免费的 Oracle Cloud 提供的 vCPU 多 2 个，但这是我能得到的最接近的设置。

## 适用于 Kubernetes 的 Oracle 云工具

首先，我不得不说，如果您使用过 AWS 或 GCP，Oracle Cloud 控制台界面是一个很大的退步。但我的意思是，如果你能省下那么多钱，谁在乎呢？

我个人在使用 Oracle Cloud Console UI 时遇到了困难，尤其是在为我的用户设置 MFA 时，但几天后就没事了。你只需要习惯它。

那么，让我们看看 Oracle Cloud 提供哪些服务来托管 Kubernetes 集群。

VPC在这里被称为VCN，Virtual Cloud Network。在那里您可以创建子网、路由表、NAT 网关、Internet 网关以及管理您的安全列表。VCN 服务始终免费，尽管进出流量有上限。我不太记得是哪一个，但我认为上限是 10 TB。

EC2 简称为计算实例。现在是好事。您将获得 2 个 AMD 微型计算实例，并且最多可以获得 4 个基于 ARM 的计算实例，每个实例具有 1 个 vCPU 和 6 GB RAM。总共**4 个 ARM vCPU 和 24 GB RAM**。是的，没错。他们总是免费的。

Kubernetes 集群称为 Kubernetes 容器引擎 (OKE)。我喜欢他们采用的方法。它是完全免费的。这是他们定价页面的报价：

> 使用 Container Engine for Kubernetes 进行集群管理不收取额外费用。客户只需为容器化工作负载使用的基础设施付费，例如工作节点（计算）、存储和其他消耗的资源。Oracle 管理多可用性域父节点并免费提供给客户。

我喜欢它。我的意思是，云提供商为 Kubernetes 集群和底层计算基础设施收费，这对我来说总是很奇怪。特别是他们收取的金额。疯狂的。

Docker 注册表称为容器注册表。此服务没有额外费用，除了图像存储费用，他们按照与对象存储（Oracle Cloud 中的 S3）相同的费率收费。但是，始终免费套餐包括 20 GB 的存储空间，这已经绰绰有余了。

负载均衡。这里有两种类型的负载均衡器，就像 AWS 一样。ALB 称为灵活负载均衡器。网络负载均衡器的名称相同。一个灵活的负载均衡器始终免费，带宽上限为 10 Mbps。单个网络负载均衡器始终是免费的，这对我来说也有点奇怪。他们的定价页面显示[无限数量的 NLB 也是免费的](https://www.oracle.com/cloud/networking/load-balancing-pricing.html)。可能只是一个错字，我不确定，但对于这个设置，我们对任何类型的单个负载均衡器都非常满意。

## 概括

这篇文章有点短，但还有更多内容。这里重要的是如果你想以零成本建立一个 Kubernetes 集群；或者至少您不想为 Kubernetes 集群每月支付 200 美元以上的费用。Oracle 云可以为您提供帮助。

请记住，由于节点是 ARM 实例，因此构建与 ARM 架构兼容的 docker 映像需要一些技巧。

接下来，我将向您展示如何在 Oracle Cloud 上使用 Terraform 设置功能齐全的 Kubernetes 集群。然后我们将检查如何为 ARM 集群构建 docker 镜像；或者事实上对于混合集群也是如此。

> 部分文章、数据、图片来自互联网,一切版权均归原网站或原作者所有。
>
> 如果侵犯了你的权益请来信告知删除。
