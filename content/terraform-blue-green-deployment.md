---
title: 使用 Terraform 和 Kubernetes 实现蓝绿部署
draft: false
tags:
  - Terraform
  - Kubernetes
  - DevOps
  - 蓝绿部署
---

在部署应用的新版本时，你应该使用一种能够将对用户的潜在影响降至最低的策略。**蓝绿部署（Blue/Green Deployments）**就是这样一个优秀的候选方案。你可以仅使用服务（Services）和部署（Deployments）等原生 Kubernetes 对象在 Kubernetes 中实现蓝绿部署，并使用 Terraform 来编排所有必需的步骤。

在这篇博客中，我们将共同学习如何使用 Terraform 在 Kubernetes 中实现蓝绿部署。

### 我们将涵盖的内容：
1. [什么是 Kubernetes？](#什么是-kubernetes)
2. [什么是蓝绿部署？](#什么是蓝绿部署)
3. [如何使用 Terraform 和 Kubernetes 实现蓝绿部署](#如何使用-terraform-和-kubernetes-实现蓝绿部署)
4. [使用 Terraform 在 Kubernetes 中进行蓝绿部署的最佳实践](#使用-terraform-在-kubernetes-中进行-蓝绿部署的最佳实践)

---

## 什么是 Kubernetes？ ☸️

[Kubernetes](https://spacelift.io/blog/kubernetes)（通常简称为 K8s）是目前容器编排领域的行业标准。它负责管理在集群中以容器形式运行的应用。Kubernetes 集群由控制平面（Control Plane）和工作节点（Worker Nodes）组成，容器会被调度并放置在这些工作节点上运行。

![Kubernetes 架构图](https://spacelift.io/wp-content/uploads/2024/07/kubernetes-diagram.png)

在 Kubernetes 上管理工作负载具有**声明式（Declarative）**的特征。你只需告诉 Kubernetes 集群你期望的状态是什么，集群就会自动负责将实际状态调整为你的期望状态。这与 Terraform 的工作方式非常相似，因此这两项技术可以完美地协同工作。

Kubernetes 是开源的，由 Linux 基金会（Linux Foundation）和云原生计算基金会（CNCF）支持。它于 2014 年首次发布，其设计灵感源自谷歌在其生产环境中运行了多年的 Borg 系统。

如今，Kubernetes 已成为大规模运行应用最受欢迎的平台之一，并且在未来很长一段时间内都将继续扮演这一重要角色。你可以浏览 CNCF 的 [云原生全景图（Cloud Native Landscape）](https://landscape.cncf.io/)，以直观感受 Kubernetes 生态系统中可用的海量工具。

从零开始搭建 Kubernetes 是一项巨大的工程，在大规模生产环境中运行自建的 Kubernetes 集群也充满挑战。不过，所有主流云厂商都提供了托管的 Kubernetes 服务：
* AWS 提供了 Elastic Kubernetes Service（EKS）
* 谷歌云提供了 Google Kubernetes Engine（GKE）
* 微软 Azure 提供了 Azure Kubernetes Service（AKS）

使用托管服务可以显著简化 Kubernetes 的入门难度，让你能够将更多精力集中在运行的应用本身，而不是集群的管理上。

### 如何使用 Terraform 管理 Kubernetes？

Kubernetes 暴露了一个功能强大的 API，允许你管理 Kubernetes 环境的方方面面。只要存在 API，通常就意味着会有相应的 Terraform 提供商（Provider）来对接该 API。对于 Kubernetes，Terraform 提供了 [Kubernetes Provider](https://spacelift.io/blog/terraform-kubernetes-provider) 和 [Helm Provider](https://spacelift.io/blog/terraform-helm-provider)。

在这篇博客中，我们将把重点放在 **Kubernetes Provider** 上。

与使用 [其他任何 Terraform Provider](https://spacelift.io/blog/terraform-providers) 一样，你必须在 Terraform 配置中声明使用 Kubernetes Provider。你需要在 `terraform` 块内的 `required_providers` 块中进行配置：

```hcl
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.37.1"
    }
  }
}
```

你还需要为 Kubernetes Provider 配置凭证以及目标集群的详细信息。

在本教程的示例中，我们将使用本地的 **Minikube** 集群。我们将配置 Provider，使其读取我本地的 Kubernetes 配置文件目录以获取认证信息和集群详情：

```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

该 Provider 提供了非常丰富的配置项。要探索所有的配置可能性，请参阅 [Kubernetes Provider 官方文档](https://registry.terraform.io/providers/hashicorp/kubernetes/latest)。

---

## 什么是蓝绿部署？ 🔵🟢

蓝绿部署使用两个完全相同的运行环境：**蓝色（Blue，当前生产环境）** 和 **绿色（Green，新版本/预发布环境）**。最初，所有流量都路由到蓝色环境。在将新版本部署到绿色环境并完成验证后，将流量一次性切换到绿色环境。如果出现任何问题，可以通过将流量重新导回蓝色环境来实现快速回滚。

![蓝绿部署工作原理图](https://spacelift.io/wp-content/uploads/2024/07/how-do-blue-green-deployments-work.png)

在 [蓝绿部署](https://spacelift.io/blog/blue-green-deployment-kubernetes) 中，你将拥有两个并排部署的环境。在任何给定的时间点，**只有一个**环境会接收实际的生产流量。初始环境被称为**蓝色环境**，运行着你当前生产版本的应用。

当需要发布新版本时，你将部署一个完全独立的**绿色环境**。起初，绿色环境不会接收任何生产流量。你可以在绿色环境中针对新版本运行各种测试，而不必担心会对生产用户产生任何影响。

一旦你对新版本的应用感到满意，就可以将生产流量从蓝色环境无缝切换到绿色环境。

在完成流量切换后，你应该让蓝色环境继续保持运行一段时间，直到你完全确信新运行的绿色环境在接收生产流量后没有出现任何问题。如果新绿色环境出现任何故障，你可以瞬间将流量切换回蓝色环境。

蓝绿部署之所以极具吸引力，是因为它们允许用户在生产环境中对新版本进行充分测试，同时又不向其发送实际的用户流量，并且在出现问题时能够实现**近乎零停机时间**的极速回滚。

与此类似的一种部署策略叫做 **金丝雀部署（Canary Deployments）**。

其核心思路与蓝绿部署类似，但在金丝雀部署中，你会首先允许极小比例的生产流量（例如 5%）到达新版本的应用。随后，逐步增加发送到新版本的流量比例，直到达到 100%。

在整个过程中，你需要密切监控新版本应用的各项指标，并在发现任何异常时立即触发自动回滚。

> 📖 **延伸阅读：** [8 种不同的 Kubernetes 部署策略](https://spacelift.io/blog/kubernetes-deployment-strategies)

---

## 如何使用 Terraform 和 Kubernetes 实现蓝绿部署 🛠️

在接下来的示例中，我们将演示如何仅使用原生 Kubernetes 对象（部署和服务）与 Terraform 来实现蓝绿部署。虽然市面上有一些第三方工具可以用来编排蓝绿部署，但掌握原生对象的实现方式能让你更深刻地理解其底层逻辑。

### 步骤 1. 准备 Kubernetes 集群 ⚙️

如前所述，我们将在本地使用 Minikube 运行 Kubernetes 集群。

如果你想跟随本教程一起动手操作，可以参考 [Minikube 官方入门文档](https://minikube.sigs.k8s.io/docs/start/) 在你的电脑上进行安装。

接下来的步骤对于任何类型的 Kubernetes 集群都是通用的。不过，访问在 Minikube 上运行的应用的细节可能会与云端集群略有不同。

### 步骤 2. 部署初始应用版本（蓝色环境） 🔵

我们首先准备初始应用版本。

在这个演示中，我们将使用一个基于 Python 编写的简单 Flask Web 应用。该应用只有一个根路径（`"/"`），并返回一个静态的文本消息。

在一个名为 `app.py` 的文件中编写以下 Python 代码：

```python
from flask import Flask

app = Flask(__name__)

@app.route("/", methods=["GET"])
def home():
    return "Spacelift App V1"

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000)
```

关于此应用，有两个关键细节需要注意：
1. 访问根路径 `"/"` 将返回静态消息 `"Spacelift App V1"`。
2. 该应用在容器内部监听 `5000` 端口。

为了安装应用所需的依赖，创建一个 `requirements.txt` 文件并写入：

```text
Flask
```

现在我们需要将该应用打包成一个 Docker 镜像，以便在 Kubernetes 集群上作为容器运行。在相同目录下创建一个名为 `Dockerfile` 的文件，内容如下：

```dockerfile
FROM python:3.9-slim-buster

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000

CMD ["python", "app.py"]
```

在终端中进入包含上述文件的目录，构建你的 Docker 镜像：

```bash
$ docker build -t spacelift-app:v1 .
```

这里我们为镜像打上了 `v1` 标签，代表我们应用的版本 1。

现在我们准备好使用 [Terraform 将应用发布到 Kubernetes](https://spacelift.io/blog/terraform-kubernetes-deployment) 了。

创建一个名为 `main.tf` 的文件，所有的 Terraform 配置都将写入其中。请确保包含我们在文章开头展示的 `terraform` 和 `provider` 块。

首先，我们添加一个名为 `spacelift` 的 Kubernetes 命名空间（Namespace）：

```hcl
resource "kubernetes_namespace" "spacelift" {
  metadata {
    name = "spacelift"
  }
}
```

在此命名空间中，我们首先创建一个 [Kubernetes 服务（Service）](https://spacelift.io/blog/kubernetes-service)，它将作为该应用的统一访问入口：

```HCL
resource "kubernetes_service" "default" {
  metadata {
    name      = "spacelift-app"
    namespace = kubernetes_namespace.spacelift.metadata[0].name
  }

  spec {
    type = "NodePort"

    selector = {
      app = "spacelift-app-v1"
    }

    port {
      port        = 80
      target_port = 5000
    }
  }
}
```

请注意，该 Kubernetes 服务使用的是标签选择器 `app = "spacelift-app-v1"`。它在 `80` 端口监听流量，并将流量转发到我们应用所监听的 `5000` 目标端口。

最后，我们添加应用版本 1 的 Kubernetes 部署（Deployment）资源：

```hcl
resource "kubernetes_deployment" "v1" {
  metadata {
    name      = "spacelift-app-v1"
    namespace = kubernetes_namespace.spacelift.metadata[0].name
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "spacelift-app-v1"
      }
    }

    template {
      metadata {
        labels = {
          app = "spacelift-app-v1"
        }
      }

      spec {
        container {
          name              = "app"
          image             = "spacelift-app:v1"
          image_pull_policy = "Never"
        }
      }
    }
  }
}
```

关于该部署，请注意以下细节：
* 应用被配置为运行 3 个实例（`replicas = 3`）。
* 它拥有必需的标签 `app = "spacelift-app-v1"`，这正是我们刚才创建的服务所寻找的。
* 它使用我们此前构建的 `spacelift-app:v1` 镜像。

我们将镜像拉取策略 `image_pull_policy` 设置为了 `Never`，因为我们希望直接使用本地电脑上构建的镜像。在实际生产场景中，你应该使用适合你环境的拉取策略。

为了让 Minikube 能够读取本地的 Docker 镜像，你需要在终端中运行以下命令以配置本地的环境变量：

```bash
$ eval $(minikube docker-env)
```

现在，我们准备好发布版本 1 了！

依次运行 `terraform init`、`terraform plan` 和 `terraform apply` 来将资源部署到 Kubernetes 集群中。部署完成后，我们可以验证 Pod 是否正常运行：

```bash
$ kubectl get pods --namespace spacelift --show-labels
NAME                               READY   STATUS    RESTARTS   AGE   LABELS
spacelift-app-v1-585ffbb5b-9krzw   1/1     Running   0          61s   app=spacelift-app-v1
spacelift-app-v1-585ffbb5b-bwspr   1/1     Running   0          61s   app=spacelift-app-v1
spacelift-app-v1-585ffbb5b-nnzvj   1/1     Running   0          61s   app=spacelift-app-v1
```

我们也可以验证 Kubernetes 服务是否已成功创建：

```bash
$ kubectl get services -n spacelift
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
spacelift-app   NodePort   10.98.41.218   <none>        80:30971/TCP   2m5s
```

在本地浏览器中访问该应用，我们可以使用 Minikube 提供的便捷命令来暴露该服务：

```bash
$ minikube service spacelift-app -n spacelift --url
http://127.0.0.1:60671
```

该命令会输出一个本地的回环地址。在浏览器中打开此地址，你将看到以下内容：

![访问部署成功的版本 1 应用](https://spacelift.io/wp-content/uploads/2025/07/blue-green-terraform-example-1.png)

第一阶段圆满成功！

### 步骤 3. 部署更新的应用版本（绿色环境） 🟢

过了一段时间，我们决定优化应用返回的文本，让它看起来更酷炫一些。

我们计划发布一个新版本，在消息中加入火箭图标 🚀。然而，我们希望在向其引入生产流量之前对其进行充分的测试。这正是蓝绿部署策略完美的演练场景！

我们首先更新 `app.py` 中的应用源码：

```python
from flask import Flask

app = Flask(__name__)

@app.route("/", methods=["GET"])
def home():
    return "Spacelift App V2 🚀"

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000)
```

保存代码后，我们使用相同的 `Dockerfile` 构建新版本的镜像：

```bash
$ docker build -t spacelift-app:v2 .
```

这一次，我们将 Docker 镜像标记为 `v2`。

接下来，我们在 `main.tf` 中**新增**一个用于新版本应用的部署资源，而不是去修改原有的部署：

```hcl
resource "kubernetes_deployment" "v2" {
  metadata {
    name      = "spacelift-app-v2"
    namespace = kubernetes_namespace.spacelift.metadata[0].name
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "spacelift-app-v2"
      }
    }

    template {
      metadata {
        labels = {
          app = "spacelift-app-v2"
        }
      }

      spec {
        container {
          name              = "app"
          image             = "spacelift-app:v2"
          image_pull_policy = "Never"
        }
      }
    }
  }
}
```

请注意关于新部署资源的以下细节：
* 这是一个**全新的资源**，我们完全没有改动已有的 `v1` 部署。在最初阶段，两个部署将并存。
* 该部署使用的标签是 `app = "spacelift-app-v2"`。
* 它使用我们新构建的 `spacelift-app:v2` 镜像。
* 我们为新部署配置了与当前生产环境完全一致的副本数（3个）。这是一个非常关键的细节，因为我们将在测试完成后一次性将流量切换过来，因此绿色环境必须具备承载全部生产流量的能力。

此时最为重要的一点是：**我们尚未修改最初创建的 Kubernetes 服务，它依然指向 v1 版本。**

运行 `terraform plan` 和 `terraform apply` 部署新资源。部署完成后，我们可以看到两个版本的应用正在并存运行：

```bash
$ kubectl get pods --namespace spacelift --show-labels
NAME                                READY   STATUS    RESTARTS   AGE     LABELS
spacelift-app-v1-585ffbb5b-9krzw    1/1     Running   0          7m40s   app=spacelift-app-v1
spacelift-app-v1-585ffbb5b-bwspr    1/1     Running   0          7m40s   app=spacelift-app-v1
spacelift-app-v1-585ffbb5b-nnzvj    1/1     Running   0          7m40s   app=spacelift-app-v1
spacelift-app-v2-65cb9757db-2cnls   1/1     Running   0          88s     app=spacelift-app-v2
spacelift-app-v2-65cb9757db-5srp8   1/1     Running   0          88s     app=spacelift-app-v2
spacelift-app-v2-65cb9757db-x5n9g   1/1     Running   0          88s     app=spacelift-app-v2
```

现在是运行所有必要测试以验证新版本是否正常工作的黄金时间。例如，为了直接测试新版本，我们可以使用端口转发（Port Forwarding）直接访问新版本的某一个 Pod：

```bash
$ kubectl port-forward spacelift-app-v2-65cb9757db-x5n9g 5000:5000 --namespace spacelift
```

在浏览器中打开 `http://127.0.0.1:5000`，我们成功看到了新版本的炫酷输出：

![验证部署成功的版本 2 应用](https://spacelift.io/wp-content/uploads/2025/07/blue-green-terraform-example-2.png)

### 步骤 4. 将生产流量切换到新版本（绿色环境） 🚀

测试完成，结果符合预期！我们准备好将生产流量从旧的蓝色环境（v1）无缝切换到新的绿色环境（v2）了。

我们只需更新 `main.tf` 中的 `kubernetes_service` 资源，将其标签选择器指向新版本应用：

```hcl
resource "kubernetes_service" "default" {
  metadata {
    name      = "spacelift-app"
    namespace = kubernetes_namespace.spacelift.metadata[0].name
  }

  spec {
    type = "NodePort"

    selector = {
      app = "spacelift-app-v2" # 将其更新为 v2
    }

    port {
      port        = 80
      target_port = 5000
    }
  }
}
```

再次运行 `terraform plan` 和 `terraform apply` 来更新服务资源。

服务更新后，刷新之前通过 Minikube 打开的生产环境网页，你将看到应用已经无缝更新为新版本：

![生产流量无缝切换到版本 2](https://spacelift.io/wp-content/uploads/2025/07/blue-green-terraform-example-3.png)

⚠️ **请牢记：此时千万不要急于删除旧的蓝色环境（v1）部署。** 你需要让它继续保持在线状态，以便在新版本在真实流量下暴露隐藏问题时，能够瞬间回滚。

### 步骤 5. 下线旧版本的应用 🗑️

经过一段时间的观察，新版本表现完美，没有任何异常。我们终于可以安全地下线旧版本的蓝色环境应用了。

从 `main.tf` 中将 `v1` 的部署资源删除（或注释掉）：

```hcl
# resource "kubernetes_deployment" "v1" {
#   metadata {
#     name      = "spacelift-app-v1"
#     namespace = kubernetes_namespace.spacelift.metadata[0].name
#   }
#   ... 其余细节省略 ...
# }
```

最后运行一次 `terraform plan` 和 `terraform apply`，Terraform 将从你的 Kubernetes 集群中优雅地移除旧的部署。完成后，验证集群中是否仅留下了新版本的 Pod：

```bash
$ kubectl get pods --namespace spacelift --show-labels
NAME                                READY   STATUS    RESTARTS   AGE   LABELS
spacelift-app-v2-65cb9757db-2cnls   1/1     Running   0          15m   app=spacelift-app-v2
spacelift-app-v2-65cb9757db-5srp8   1/1     Running   0          15m   app=spacelift-app-v2
spacelift-app-v2-65cb9757db-x5n9g   1/1     Running   0          15m   app=spacelift-app-v2
```

大功告成！我们已经使用 Terraform 在 Kubernetes 中完美实现了一次蓝绿部署。

> 💡 **你可能也会喜欢：**
> * [Terraform 使用的 20 个最佳实践](https://spacelift.io/blog/terraform-best-practices)
> * [大规模管理 Terraform 的 5 种方法](https://spacelift.io/blog/5-ways-to-manage-terraform-at-scale)
> * [如何实现 Terraform 的自动化部署](https://spacelift.io/blog/terraform-automation)

---

## 使用 Terraform 在 Kubernetes 中进行蓝绿部署的最佳实践 🌟

为了确保你的蓝绿部署安全、高效地运行，请参考以下最佳实践：

### 1. 自动化蓝绿部署流程 🤖

在这篇博客中，为了清晰展示底层原理，我们手动一步步执行了蓝绿部署。

但在实际的生产环境中，**你必须将这一过程完全自动化**。所有手动执行的步骤都可以轻松用脚本完成。最简单的方式是编写一个自动化脚本来按顺序执行这些步骤。当然，更佳的方案是利用你现有的 CI/CD 系统或基础设施即代码（IaC）协作平台（如 Spacelift）来编排整个工作流。

自动化不仅能最大程度减少人为操作失误，还能在遇到异常时自动触发秒级回滚。

### 2. 实施持续测试与反馈机制 🧪

你应该持续优化你的部署流程，并引入全面的测试，以增强你对蓝绿部署的信心。

如果在某次部署中遇到了问题，不要只是简单地回滚了事。你应该仔细分析根本原因（Post-Mortem），找出流程中的漏洞，并优化流程以防其再次发生。

测试的类型应根据你的具体应用场景来设计。至少，你应该拥有一套端到端（E2E）测试套件，用以模拟真实用户在你的应用上进行的核心操作流程。

### 3. 准备周密且经过验证的回滚计划 📋

即使你对新版本运行了海量的测试，在真实的用户流量涌入时，依然可能会暴露出意想不到的惊喜（或惊吓）。

为了应对这种情况，你必须随时准备好一份**自动化的回滚方案**。这意味着如果新版本的监控指标出现异常，你可以通过一行命令或自动化流水线，将 Kubernetes 服务立即指向旧版本的部署。

### 4. 明确定义并量化“部署成功”的指标 📊

要想知道何时应该触发回滚，你必须先明确什么才是“正常的生产状态”。

你应该在部署前后密切监控应用的关键绩效指标（KPIs）。如果任何 KPI 指标出现负面下滑，都应果断触发回滚。

典型的监控指标包括：
* HTTP 状态码（如 5xx 错误率）
* 接口响应延迟
* CPU 和内存的利用率
* 业务转化率等

### 5. 极其谨慎地处理状态应用（Stateful Applications） 💾

我们在本教程中展示的是无状态应用（Stateless Application）的蓝绿部署。但对于包含数据库或其他持久化存储的状态应用，蓝绿部署要复杂得多。

如果新版本应用包含数据库表结构的变更（Schema Changes），你必须采用多阶段过渡的策略。

例如，如果你的应用使用 PostgreSQL 数据库，并需要修改某张数据库表。你应当遵循以下部署步骤：
1. 部署绿色版本的应用，但不要立即切换流量。
2. 对数据库进行**向前和向后兼容**的更新。这意味着在切换流量前，旧的蓝色版本依然能够无缝读写更新后的数据库表。
3. 将流量切换到绿色版本。
4. 观察无误后，下线旧的蓝色版本。
5. （可选）应用数据库表结构的进一步优化或清理变更（此时无需再兼顾旧版本）。

对于某些非常重大的变更，你甚至可能需要将上述过程拆分为多个微小的迭代步骤来分批实施。

---

## 如何使用 Spacelift 管理 Kubernetes 和 Terraform 🚀

Spacelift 深度支持 [Terraform](https://docs.spacelift.io/vendors/terraform/) 和 [Kubernetes](https://docs.spacelift.io/vendors/kubernetes/)（以及许多其他 IaC 工具），并允许用户以此为基础构建 **堆栈（Stacks）**。

借助 Spacelift，你可以构建统一的 CI/CD 流水线，将这两者深度结合，充分发挥每个工具的最大优势。通过这种方式，你可以使用单一入口来管理 Terraform 和 Kubernetes 资源的完整生命周期，促进团队的高效协作，并为你的日常工作流注入必不可少的安全合规控制。

![Spacelift 管理 Kubernetes 和 Terraform 的堆栈依赖](https://spacelift.io/wp-content/uploads/2025/07/stack-dependencies-kubernetes-terraform.png)

例如，你可以使用 Terraform 堆栈来预配底层 Kubernetes 集群，然后在独立的 Kubernetes 堆栈上，将容器化应用发布到该集群中。这种方式还能让你极其轻松地将**漂移检测（Drift Detection）**集成到你的 Kubernetes 堆栈中，实现基础设施状态的主动防御。

> 🔗 想了解为何结合使用 Kubernetes、Terraform 和 Spacelift 是最明智的选择？请阅读这篇 [ArgoCD 与 Terraform 的对比分析文章](https://spacelift.io/blog/argocd-terraform#managing-terraform--kubernetes-with-spacelift)。本教程的相关源码也可在 [GitHub 仓库](https://github.com/saturnhead/eks-argo-terraform/tree/main) 中找到。

如果你想亲身体验 Spacelift 的强大功能，欢迎 [注册免费试用账号](https://spacelift.io/free-trial) 或 [预约我们的工程师进行演示演示（Demo）](https://spacelift.io/schedule-demo)。

---

## 关键要点总结 📝

* **蓝绿部署**是向生产环境安全引入新版本应用的卓越策略，能提供充分的测试空间和秒级回滚保障。
* **使用 Terraform 在 Kubernetes 中实现蓝绿部署的四个核心阶段：**
  1. 部署初始（蓝色）应用版本，配置对应的命名空间、服务和部署。这需要第一次 `terraform apply`。
  2. 部署新（绿色）应用版本，在同一命名空间中作为完全独立的部署运行。这需要第二次 `terraform apply`。
  3. 完成新版本验证后，更新 Kubernetes 服务，使其 Selector 指向新的部署。这需要第三次 `terraform apply`。
  4. 确认新版本稳定运行后，下线并清理旧的（蓝色）部署资源。这需要第四次也是最后一次 `terraform apply`。
* 蓝绿部署的成功高度依赖于**流程自动化**、**持续测试**、**明确的成功指标量化**以及**状态应用数据的兼容性处理**。

> 💡 **小贴士：** 尽管 HashiCorp 对新版本的 Terraform 采用了 BUSL 协议，但 1.5.x 版本之前的 Terraform 依然保持开源。作为 Linux 基金会旗下的开源项目，[OpenTofu](https://opentofu.org/) 是一个完全兼容且极具活力的开源 Terraform 替代方案，为你提供了更广阔的云原生选择。

### 让基础设施管理变得更简单
Spacelift 能够极其高效地管理 Terraform 状态、编排高度复杂的流水线、支持“策略即代码”（Policy-as-Code）、提供漂移检测及可视化拓扑，是 DevOps 团队不可或缺的强大助力。[了解更多关于 Terraform 自动化的信息](/terraform-automation)
