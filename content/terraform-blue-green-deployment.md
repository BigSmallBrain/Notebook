---
title: 使用 Terraform 和 Kubernetes 实现蓝绿部署
draft: false
tags:
  - Terraform
  - Kubernetes
  - DevOps
  - 蓝绿部署
---

在部署应用的新版本时，应该使用一种能够将对用户的潜在影响降至最低的策略。**蓝绿部署（Blue/Green Deployments）就是这样一个优秀的候选方案。你可以仅使用服务（Services）和部署（Deployments）等原生 Kubernetes 对象在 Kubernetes 中实现蓝绿部署，并使用 Terraform 来编排所有必需的步骤。

在这篇博客中，将介绍如何使用 Terraform 在 Kubernetes 中实现蓝绿部署。

---

## 什么是 Kubernetes？ ☸️

[Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/overview/)（通常简称为 K8s）是目前容器编排领域的行业标准。它负责管理在集群中以容器形式运行的应用。Kubernetes 集群由控制平面（Control Plane）和工作节点（Worker Nodes）组成，容器会被调度并放置在这些工作节点上运行。

![[Pasted image 20260531211406.png]]

在 Kubernetes 上管理工作负载具有**声明式（Declarative）的特征。你只需告诉 Kubernetes 集群你期望的状态是什么，集群就会自动负责将实际状态调整为你的期望状态。这与 Terraform 的工作方式非常相似，因此这两项技术可以完美地协同工作。
### 如何使用 Terraform 管理 Kubernetes？

Kubernetes 暴露了一个功能强大的 API，允许你管理 Kubernetes 环境的方方面面。只要存在 API，通常就意味着会有相应的 Terraform 提供商（Provider）来对接该 API。对于 Kubernetes，Terraform 提供了 [Kubernetes Provider](https://registry.terraform.io/providers/hashicorp/kubernetes/2.37.1) 。

与使用其他任何 Terraform Provider 一样，你必须在 Terraform 配置中声明使用 Kubernetes Provider。你需要在 `terraform` 块内的 `required_providers` 块中进行配置：

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

在本教程的示例中使用本地的 **Minikube** 集群。将配置 Provider，使其读取本地的 Kubernetes 配置文件目录以获取认证信息和集群详情：

```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

该 Provider 提供了非常丰富的配置项。要探索所有的配置可能性，请参阅 [Kubernetes Provider 官方文档](https://registry.terraform.io/providers/hashicorp/kubernetes/2.37.1)。

---

## 什么是蓝绿部署？ 🔵🟢

蓝绿部署使用两个完全相同的运行环境：**蓝色（Blue，当前生产环境）** 和 **绿色（Green，新版本/预发布环境）**。最初，所有流量都路由到蓝色环境。在将新版本部署到绿色环境并完成验证后，将流量一次性切换到绿色环境。如果出现任何问题，可以通过将流量重新导回蓝色环境来实现快速回滚。

![[Pasted image 20260531212800.png]]

在蓝绿部署中，有两个并排部署的环境。在任何给定的时间点，**只有一个**环境会接收实际的生产流量。初始环境被称为**蓝色环境**，运行着你当前生产版本的应用。

当需要发布新版本时，将部署一个完全独立的**绿色环境**。起初，绿色环境不会接收任何生产流量。可以在绿色环境中针对新版本运行各种测试，而不必担心会对生产用户产生任何影响。

一旦对新版本的应用感到满意，就可以将生产流量从蓝色环境无缝切换到绿色环境。

在完成流量切换后，应该让蓝色环境继续保持运行一段时间，直到你完全确信新运行的绿色环境在接收生产流量后没有出现任何问题。如果新绿色环境出现任何故障，你可以瞬间将流量切换回蓝色环境。

蓝绿部署之所以极具吸引力，是因为它们允许用户在生产环境中对新版本进行充分测试，同时又不向其发送实际的用户流量，并且在出现问题时能够实现**近乎零停机时间**的极速回滚。

与此类似的一种部署策略叫做 **金丝雀部署（Canary Deployments）**。

其核心思路与蓝绿部署类似，但在金丝雀部署中，会首先允许极小比例的生产流量（例如 5%）到达新版本的应用。随后，逐步增加发送到新版本的流量比例，直到达到 100%。

在整个过程中需要密切监控新版本应用的各项指标，并在发现任何异常时立即触发自动回滚。

---

## 如何使用 Terraform 和 Kubernetes 实现蓝绿部署 🛠️

在接下来的示例中，我们将演示如何仅使用原生 Kubernetes 对象（部署和服务）与 Terraform 来实现蓝绿部署。虽然市面上有一些第三方工具可以用来编排蓝绿部署，但掌握原生对象的实现方式能让你更深刻地理解其底层逻辑。

### 步骤 1. 准备 Kubernetes 集群 ⚙️

如前所述，接下来将在本地使用 Minikube 运行 Kubernetes 集群，可以参考 [Minikube 官方入门文档](https://minikube.sigs.k8s.io/docs/start/) 进行安装。

接下来的步骤对于任何类型的 Kubernetes 集群都是通用的。不过，访问在 Minikube 上运行的应用的细节可能会与云端集群略有不同。

### 步骤 2. 部署初始应用版本（蓝色环境） 🔵

首先准备初始应用版本。

在这个示例中，将使用一个基于 Python 编写的简单 Flask Web 应用。该应用只有一个根路径（`"/"`），并返回一个静态的文本消息。

在一个名为 `app.py` 的文件中编写以下 Python 代码：

```python
from flask import Flask

app = Flask(__name__)

@app.route("/", methods=["GET"])
def home():
    return "Kingslayer App V1 🚀"

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000)
```

关于此应用，有两个关键细节需要注意：
1. 访问地址 `"/"` 将返回静态消息 `"Kingslayer App V2 🚀"`。
2. 该应用在容器内部监听 `5000` 端口。

为了安装应用所需的依赖，创建一个 `requirements.txt` 文件并写入：

```text
Flask
```

现在需要将该应用打包成一个 Docker 镜像，以便在 Kubernetes 集群上作为容器运行。在相同目录下创建一个名为 `Dockerfile` 的文件，内容如下：

```dockerfile
FROM python:3.9-slim-buster

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app
EXPOSE 5000
CMD ["python", "app/app.py"]
```

在终端中进入包含上述文件的目录，构建你的 Docker 镜像：

```bash
docker build -t kingslayer-app:v1 .
```

这里的镜像打上了 `v1` 标签，代表应用的版本为 v1。

创建一个名为 `main.tf` 的文件，所有的 Terraform 配置都将写入其中。这里包含上述在文章开头展示的 `terraform` 和 `provider` 块。

首先添加一个名为 `kingslayer` 的 Kubernetes 命名空间（Namespace）：

```hcl
resource "kubernetes_namespace" "kingslayer" {
  metadata {
    name = "kingslayer"
  }
}
```

在此命名空间中，首先创建一个 Kubernetes 服务（Service），它将作为该应用的统一访问入口：

```hcl
resource "kubernetes_service" "default" {
  metadata {
    name      = "kingslayer-app"
    namespace = kubernetes_namespace.kingslayer.metadata[0].name
  }

  spec {
    type = "LoadBalancer"

    selector = {
      app = "kingslayer-app-v1"
    }

    port {
      port        = 5000
      target_port = 5000
    }
  }
}
```

请注意，该 Kubernetes 服务使用的是标签选择器 `app = "kingslayer-app-v1"`。它在 `5000` 端口监听流量，并将流量转发到应用所监听的 `5000` 目标端口。

最后，添加应用版本 v1 的 Kubernetes 部署（Deployment）资源：

```hcl
resource "kubernetes_deployment" "v1" {
  metadata {
    name      = "kingslayer-app-v1"
    namespace = kubernetes_namespace.kingslayer.metadata[0].name
  }

  spec {
    replicas = 2

    selector {
      match_labels = {
        app = "kingslayer-app-v1"
      }
    }

    template {
      metadata {
        labels = {
          app = "kingslayer-app-v1"
        }
      }

      spec {
        container {
          name              = "app"
          image             = "kingslayer-app:v1"
          image_pull_policy = "IfNotPresent"
        }
      }
    }
  }
}
```

关于该部署，请注意以下细节：
* 应用被配置为运行 2 个实例（`replicas = 2`）。
* 拥有必需的标签 `app = "kingslayer-app-v1"`。
* 使用此前构建的 `kingslayer-app:v1` 镜像。

将镜像拉取策略 `image_pull_policy` 设置为了 `IfNotPresent`，直接使用本地电脑上构建的镜像。在实际生产场景中，应该使用适合环境的拉取策略。

依次运行 `terraform init`、`terraform plan` 和 `terraform apply` 来将资源部署到 Kubernetes 集群中。部署完成后，验证 Pod 是否正常运行：

```bash
$ kubectl get pods --namespace kingslayer --show-labels
NAME                                 READY   STATUS    RESTARTS   AGE   LABELS
kingslayer-app-v1-7486f68f4d-4l96k   1/1     Running   0          14s   app=kingslayer-app-v1,pod-template-hash=7486f68f4d
kingslayer-app-v1-7486f68f4d-87t7d   1/1     Running   0          14s   app=kingslayer-app-v1,pod-template-hash=7486f68f4d
```

也可以验证 Kubernetes 服务是否已成功创建：

```bash
$ kubectl get services -n kingslayer
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kingslayer-app   LoadBalancer   10.96.199.53   172.18.0.3    5000:30456/TCP   50s
```

在浏览器中打开此地址 `http://127.0.0.1:5000/`，将看到以下内容：

![[Pasted image 20260531215355.png]]

第一阶段圆满成功！

### 步骤 3. 部署更新的应用版本（绿色环境） 🟢

过了一段时间，客户决定优化应用返回的文本，让它看起来更酷炫一些。

计划发布一个新版本，在消息中多加入一个火箭图标 🚀。然而，客户希望在向其引入生产流量之前对其进行充分的测试。这正是蓝绿部署策略完美的演练场景！

首先更新 `app.py` 中的应用源码：

```python
from flask import Flask

app = Flask(__name__)

@app.route("/", methods=["GET"])
def home():
    return "Kingslayer App V2 🚀🚀"

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000)
```

保存代码后，使用相同的 `Dockerfile` 构建新版本的镜像：

```bash
docker build -t kingslayer-app:v2 .
```

这一次，将 Docker 镜像标记为 `v2`。

接下来在 `main.tf` 中**新增**一个用于新版本应用的部署资源，而不是去修改原有的部署：

```hcl
resource "kubernetes_deployment" "v2" {
  metadata {
    name      = "kingslayer-app-v2"
    namespace = kubernetes_namespace.kingslayer.metadata[0].name
  }

  spec {
    replicas = 2

    selector {
      match_labels = {
        app = "kingslayer-app-v2"
      }
    }

    template {
      metadata {
        labels = {
          app = "kingslayer-app-v2"
        }
      }

      spec {
        container {
          name              = "app"
          image             = "kingslayer-app:v2"
          image_pull_policy = "IfNotPresent"
        }
      }
    }
  }
}
```

请注意关于新部署资源的以下细节：
* 这是一个**全新的资源**，完全没有改动已有的 `v1` 部署。在最初阶段，两个部署将并存。
* 该部署使用的标签是 `app = "kingslayer-app-v2"`。
* 它使用我们新构建的 `kingslayer-app:v2` 镜像。
* 为新部署配置了与当前生产环境完全一致的副本数（2个）。这是一个非常关键的细节，因为在测试完成后一次性将流量切换过来，因此绿色环境必须具备承载全部生产流量的能力。

此时最为重要的一点是：**没有修改最初创建的 Kubernetes 服务，它依然指向 v1 版本。**

运行 `terraform plan` 和 `terraform apply` 部署新资源。部署完成后可以看到两个版本的应用正在并存运行：

```bash
$ kubectl get pods --namespace kingslayer --show-labels
NAME                                 READY   STATUS    RESTARTS   AGE    LABELS
kingslayer-app-v1-7486f68f4d-6k6bd   1/1     Running   0          32s    app=kingslayer-app-v1,pod-template-hash=7486f68f4d
kingslayer-app-v1-7486f68f4d-gl45t   1/1     Running   0          32s    app=kingslayer-app-v1,pod-template-hash=7486f68f4d
kingslayer-app-v2-6dbbd77476-nqj95   1/1     Running   0          100s   app=kingslayer-app-v2,pod-template-hash=6dbbd77476
kingslayer-app-v2-6dbbd77476-qx8nv   1/1     Running   0          100s   app=kingslayer-app-v2,pod-template-hash=6dbbd77476
```

现在是运行所有必要测试以验证新版本是否正常工作的黄金时间。例如，为了直接测试新版本，使用端口转发（Port Forwarding）直接访问新版本的某一个 Pod：

```bash
$ kubectl port-forward kingslayer-app-v2-6dbbd77476-nqj95 4999:5000 --namespace kingslayer
```

在浏览器中打开 `http://127.0.0.1:4999`，成功看到了新版本的炫酷输出：

![[Pasted image 20260531220459.png]]

### 步骤 4. 将生产流量切换到新版本（绿色环境） 🚀

测试完成，结果符合预期！准备好将生产流量从旧的蓝色环境（v1）无缝切换到新的绿色环境（v2）了。

只需更新 `main.tf` 中的 `kubernetes_service` 资源，将其标签选择器指向新版本应用：

```hcl
resource "kubernetes_service" "default" {
  metadata {
    name      = "kingslayer-app"
    namespace = kubernetes_namespace.kingslayer.metadata[0].name
  }

  spec {
    type = "LoadBalancer"

    selector = {
      app = "kingslayer-app-v2" # 将其更新为 v2
    }

    port {
      port        = 5000
      target_port = 5000
    }
  }
}
```

再次运行 `terraform plan` 和 `terraform apply` 来更新服务资源。

服务更新后，刷新之前通过 Minikube 打开的生产环境网页，你将看到应用已经无缝更新为新版本：

![[Pasted image 20260531220748.png]]

⚠️ **请牢记：此时千万不要急于删除旧的蓝色环境（v1）部署。** 你需要让它继续保持在线状态，以便在新版本在真实流量下暴露隐藏问题时，能够瞬间回滚。

### 步骤 5. 下线旧版本的应用 🗑️

经过一段时间的观察，新版本表现完美，没有任何异常。可以安全地下线旧版本的蓝色环境应用了。

从 `main.tf` 中将 `v1` 的部署资源删除（或注释掉）：

```hcl
# resource "kubernetes_deployment" "v1" {
#   metadata {
#     name      = "kingslayer-app-v1"
#     namespace = kubernetes_namespace.kingslayer.metadata[0].name
#   }
#   ... 其余细节省略 ...
# }
```

最后运行一次 `terraform plan` 和 `terraform apply`，Terraform 将从你的 Kubernetes 集群中优雅地移除旧的部署。完成后，验证集群中是否仅留下了新版本的 Pod：

```bash
$ kubectget pods --namespace kingslayer --show-labels
NAME                                 READY   STATUS    RESTARTS   AGE     LABELS
kingslayer-app-v2-6dbbd77476-nqj95   1/1     Running   0          9m17s   app=kingslayer-app-v2,pod-template-hash=6dbbd77476
kingslayer-app-v2-6dbbd77476-qx8nv   1/1     Running   0          9m17s   app=kingslayer-app-v2,pod-template-hash=6dbbd77476
```

大功告成！已经使用 Terraform 在 Kubernetes 中完美实现了一次蓝绿部署。

---

## 使用 Terraform 在 Kubernetes 中进行蓝绿部署的最佳实践 🌟

为了确保蓝绿部署安全、高效地运行，请参考以下最佳实践：

### 1. 自动化蓝绿部署流程 🤖

在这篇博客中，为了清晰展示底层原理，我们手动一步步执行了蓝绿部署。

但在实际的生产环境中，**你必须将这一过程完全自动化**。所有手动执行的步骤都可以轻松用脚本完成。最简单的方式是编写一个自动化脚本来按顺序执行这些步骤。当然，更佳的方案是利用你现有的 CI/CD 系统或基础设施即代码（IaC）协作平台来编排整个工作流。

自动化不仅能最大程度减少人为操作失误，还能在遇到异常时自动触发秒级回滚。

### 2. 实施持续测试与反馈机制 🧪

应持续优化部署流程，并引入全面的测试，以增强对蓝绿部署的信心。

如果在某次部署中遇到了问题，不要只是简单地回滚了事。应该仔细分析根本原因（Post-Mortem），找出流程中的漏洞，并优化流程以防其再次发生。

测试的类型应根据你的具体应用场景来设计。至少，你应该拥有一套端到端（E2E）测试套件，用以模拟真实用户在你的应用上进行的核心操作流程。

### 3. 准备周密且经过验证的回滚计划 📋

即使对新版本运行了海量的测试，在真实的用户流量涌入时，依然可能会暴露出意想不到的惊喜。

为了应对这种情况，必须随时准备好一份**自动化的回滚方案**。这意味着如果新版本的监控指标出现异常，可以通过一行命令或自动化流水线，将 Kubernetes 服务立即指向旧版本的部署。

### 4. 明确定义并量化“部署成功”的指标 📊

要想知道何时应该触发回滚，必须先明确什么才是“正常的生产状态”。

应该在部署前后密切监控应用的关键绩效指标（KPIs）。如果任何 KPI 指标出现负面下滑，都应果断触发回滚。

典型的监控指标包括：
* HTTP 状态码（如 5xx 错误率）
* 接口响应延迟
* CPU 和内存的利用率
* 业务转化率等

### 5. 极其谨慎地处理状态应用（Stateful Applications） 💾

在本教程中展示的是无状态应用（Stateless Application）的蓝绿部署。但对于包含数据库或其他持久化存储的状态应用，蓝绿部署要复杂得多。

如果新版本应用包含数据库表结构的变更（Schema Changes），必须采用多阶段过渡的策略。

例如，如果你的应用使用 PostgreSQL 数据库，并需要修改某张数据库表。应当遵循以下部署步骤：
1. 部署绿色版本的应用，但不要立即切换流量。
2. 对数据库进行**向前和向后兼容**的更新。这意味着在切换流量前，旧的蓝色版本依然能够无缝读写更新后的数据库表。
3. 将流量切换到绿色版本。
4. 观察无误后，下线旧的蓝色版本。
5. （可选）应用数据库表结构的进一步优化或清理变更（此时无需再兼顾旧版本）。

对于某些非常重大的变更，甚至可能需要将上述过程拆分为多个微小的迭代步骤来分批实施。

---
## 关键要点总结 📝

- **蓝绿部署**是向生产环境安全引入新版本应用的卓越策略，能提供充分的测试空间和秒级回滚保障。
- **使用 Terraform 在 Kubernetes 中实现蓝绿部署的四个核心阶段：**
    1. 部署初始（蓝色）应用版本，配置对应的命名空间、服务和部署。这需要第一次 `terraform apply`。
    2. 部署新（绿色）应用版本，在同一命名空间中作为完全独立的部署运行。这需要第二次 `terraform apply`。
    3. 完成新版本验证后，更新 Kubernetes 服务，使其 Selector 指向新的部署。这需要第三次 `terraform apply`。
    4. 确认新版本稳定运行后，下线并清理旧的（蓝色）部署资源。这需要第四次也是最后一次 `terraform apply`。
- 蓝绿部署的成功高度依赖于**流程自动化**、**持续测试**、**明确的成功指标量化**以及**状态应用数据的兼容性处理**。

> 参考来源：[Blue/Green Deployments With Terraform & Kubernetes](https://spacelift.io/blog/terraform-blue-green-deployment)
