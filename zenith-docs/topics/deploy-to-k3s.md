# Rust 部署到 k3s

k3s 是轻量级的 K8s，比较适合中小型的项目，生产可用。这篇文档描述了使用 k3s 的一些生产实践。

## 架构 ##

对于任何技术来说，首先从宏观层面来了解其架构师非常必要的。我们先从 k3s 的架构说起:

![45916f01c59d1fc2bf280c4ef5fcb32a.png](https://i.miji.bid/2025/05/02/45916f01c59d1fc2bf280c4ef5fcb32a.png)

整体架构分为两部分，k3s Server 和 k3s Agent:

k3s Server 由如下组建构成：

* **Kine**：它是随着 k3s 一起发布的，兼容 Etcd v3 协议，用于屏蔽底层数据库之间的差异、统一转发的职责，作为数据库抽象层使用。默认情况下，使用 Sqlite；

* **API Server**：认证（Authentication）、鉴权（Authorization）以及准入（Admission，Mutating & Valiating），提供 REST API，这是 Control Panl 中回一个可由用户访问 API 进行交互的组件，提供其他模块之间的数据交互和通讯的枢纽。

* **Controller Manager**：用来管理各种 Controller，Watch 状态的变更，对比期望状态（Desired State）与实际状态（Actual State），通过创建、更新、删除资源来推动集群往期望状态靠拢。

    * 是多个控制器的组合，每个 Controller 事实上都是一个 Control loop，负责监听其管控的对象，当每个对象发生变更时完成配置；

    * 如果配置失败会触发自动重试，整个集群会在控制器不断重试机制下确保最终一致性（Eventual Consistency）。

* **Scheduler**: 调度器会监控新建的 pods，并将其分配给合适的节点；

    * 它是一个特殊的 Controller，工作原理和其他 Controller 是一样的；

    * 它的职责是监控集群中的所有没有调度的 Pod，并且获取当前集群中所有节点的健康状态和资源使用晴空，为调度 Pod 选择最佳计算节点，完成调度；

    * 分为三个阶段：**Predict**阶段过滤不满足业务需求的节点，比如资源不足、端口冲突等。**Priority**：按既定要素将满足调度需求的节点评分，选择最佳节点。**Bind**将计算节点和 Pod 进行绑定，完成调度。

k3s Agent 由如下组建构建：

* **Kubelet**：节点中 Pod 的生命周期的管理，执行任务并将 Pod 的状态报告给主节点的渠道，通过容器运行时来运行容器，并进行健康监测;

* **Flannel**：轻量级的容器网络插件，为集群中的 Pod 提供扁平化的覆盖网络；

其他的一些组件:

* **Etcd**：它是 CoreOS 基于 Raft 开发的分布式 key-value 存储系统，可用于服务发现、共享配置以及一致性保障（比如说数据库选主、分布式锁等）。

## 安装 ##

K3s 不同于 K8s，最先感受到的就是安装部署上的简单快速，单机环境下，就一行命令搞定:
```shell
## 关闭防火墙
$ systemctl disable firewalld --now
## 安装
$ curl -sfL https://get.k3s.io | sh -
```
在网络顺畅的情况下，一分钟内可以搞定。然后运行下面的命令查看节点，如果正常输出，则表示安装成功:
```shell
$ kubectl get node
NAME   STATUS   ROLES                  AGE   VERSION
k3s    Ready    control-plane,master   98s   v1.32.3+k3s1
```

## 全局 TLS 证书管理 ##

首先我们需要安装 Cert-Manager，用作全局证书管理。包括，下文中，我们需要构建私有的 Registry，也需要绑定 TLS 证书。

### 1. 安装 Cert-Manager ###

Cert-Manager 可以用来管理 k3s 集群中自动化所有的 TLS。 应用下面的配置，最新稳定版的版本号，在 GitHub Releases 页面查看:
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
```
然后通过下面的命令，验证是否安装成功:
```shell
$ kubectl -n cert-manager get pods
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-6468fc8f56-6jpd2             1/1     Running   0          3m22s
cert-manager-cainjector-7fd85dcc7-8kcrm   1/1     Running   0          3m22s
cert-manager-webhook-57df45f686-vphs8     1/1     Running   0          3m22s
```

### 2. 创建 ClusterIssuer ###

Cert-Manager 的作用是定义做什么，而 ClusterIssuer 的作用则是定义如何做，用哪一家 CA、有哪些参数，并且它是集群级别的，所有的命令空间都可以使用。

接着创建 Let's Encrypt ClusterIssuer，创建文件 `cluster-issuer.yaml`：
```shell
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: happy.hacking.icu@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
```

> Traefik 是现代化的云原生边缘代理（Edge Proxy）和 Kubernerts Ingress Controller，主要功能包括反向代理（Reverse Proxy）、负载均衡（Load Balancing）、动态服务发现（Discovery）、内置了 Let's Encrypt ACME 证书管理、支持重定向、限流、熔断、身份验证等中间件（Middleware）。在 k3s 中自动启用。

然后应用配置：
```shell
$ kubectl apply -f cluster-issuer.yaml
clusterissuer.cert-manager.io/letsencrypt-prod created
```

## 启用 Registry ##

安装后，我们首先启用镜像仓库，并且使用 Let' Encrypt 来申请 SSL 证书。在开始之前，先确保已经做好了域名解析。

### 1. 部署 Registry ####

接着部署 Registry, 创建文件 `registry-deployment.yaml`：
```shell
apiVersion: v1
kind: Namespace
metadata:
  name: registry

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
        volumeMounts:
        - name: data
          mountPath: /var/lib/registry
      volumes:
      - name: data
        hostPath:
          path: /opt/registry
          type: DirectoryOrCreate

---
apiVersion: v1
kind: Service
metadata:
  name: registry-svc
  namespace: registry
spec:
  type: ClusterIP
  ports:
  - port: 5000
    targetPort: 5000
  selector:
    app: registry
```
然后应用配置：
```shell
$ kubectl apply -f registry-deployment.yaml
namespace/registry created
deployment.apps/registry created
service/registry-svc create
```

### 2. 绑定证书 {id="registry-ssl"}

接着创建 Ingress 并绑定 Let’s Encrypt 证书，创建文件 `registry-ingress.yaml`：
```shell
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: registry-ingress
  namespace: registry
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - <你的域名>
    secretName: registry-tls
  rules:
  - host: <你的域名>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: registry-svc
            port:
              number: 5000
```
然后应用配置：
```shell
$ kubectl apply -f registry-ingress.yaml
ingress.networking.k8s.io/registry-ingress created
```

### 3. 配置私有仓库 {id="registry-config"}

配置 k3s 使用私有的仓库, 创建或编译 `/etc/rancher/k3s/registries.yaml`：
```shell
mirrors:
  "registry.k3s.testing.icu":
    endpoint:
      - "https://registry.k3s.testing.icu"
configs:
  "registry.k3s.teting.icu":
    tls:
      insecure_skip_verify: false
```
然后重启 k3s 的服务:
```shell
systemctl restart k3s
```
然后就可以通过访问 `https://<你的域名>/v2/_catalog` 来验证是否生效了。

### 4. 启用鉴权 ###

如果是在公网上访问，那么就必须添加一层鉴权，防止资源可以公开访问。

首先，生成 htpasswd 文件:
```shell
# 确保安装了: dnf install httpd-tools
$ mkdir /opt/auth
$ htpasswd -Bbn <用户名> <密码> >> /opt/auth/htpasswd
# 随后会要求输入密码
```

接着，把 htpasswd 存到 Kubernetes Secret:
```shell
$ kubectl create secret generic registry-htpasswd \
    --from-file=htpasswd=/opt/auth/htpasswd \
    -n registry
secret/registry-htpasswd created
```
修改 `registry-deployment.yaml` 配置文件，添加 Auth 相关配置:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: registry
spec:
  template:
    spec:
      containers:
      - name: registry
        image: registry:2
        env:
          - name: REGISTRY_AUTH
            value: "htpasswd"
          - name: REGISTRY_AUTH_HTPASSWD_REALM
            value: "Registry Realm"
          - name: REGISTRY_AUTH_HTPASSWD_PATH
            value: "/auth/htpasswd"
        volumeMounts:
          - name: auth
            mountPath: /auth
            readOnly: true
      volumes:
        - name: auth
          secret:
            secretName: registry-htpasswd
```
然后应用配置:
```shell
$ kubectl apply -f registry-deployment.yaml
```

然后可以通过 `docker login <域名>` 来验证是否可以 Login Succeeded。

## Hello World ##

接下来，通过一个 Hello World 的示例来展示如何将一个 Web 应用通过 GitHub CI 集成到 k3s 集群，并对外提供服务。

### 1. 编写代码 ###

这个项目以来 Tokio 和 Axum 两个框架，依赖如下：
```
[dependencies]
tokio ={ version = "1.44.2", features = ["full"] }
axum = "0.8.4"
```
项目代码如下：
```rust
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app: Router = Router::new().route("/", get(|| async { "Hello World" }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:80").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```
然后本地可以通过 `cargo run` 来测试，访问 `http://127.0.0.1` 测试是否输出 `Hello World`。

### 2. 编写 Dockerfile ###

然后为这个项目编写 Dockerfile 来创建镜像：
```rust
FROM rust:1.86-bullseye AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bullseye-slim AS runtime
WORKDIR /app
COPY --from=builder /app/target/release/app /app/app

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates \
 && rm -rf /var/lib/apt/lists/*

EXPOSE 80
CMD ["/app/app"]
```

### 3. 发布镜像 {id="publish-image"}

接下来，要发布镜像到我们之前创建的 Registry 中:
```shell
docker build -t k3s-rust-demo:1.0.2 .
docker tag app:1.0.0 <registry-domain>/app:1.0.2
docker push <registry-domain>/app:1.0.2
```

### 4. 创建命名空间 {id="create-namespace"}

接着，我们要部署应用到集群中了。首先创建命名空间:
```yaml
apiVersion: v1
kind: Namespace
metadata:
    name: k3s-rust-demo
```
应用配置文件:
```yaml
$ kubectl apply -f namespace.yaml
```

### 5. 创建部署清单 {id="create-deployment"}

接着编写 `deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: k3s-rust-demo
    labels:
        app: k3s-rust-demo
spec:
    replicas: 2
    selector:
        matchLabels:
            app: k3s-rust-demo
    template:
        metadata:
            labels:
                app: k3s-rust-demo
        spec:
            imagePullSecrets:
                - name: regcred
            containers:
                - name: k3s-rust-demo
                  image: <registry-domain>/k3s-rust-demo:0.1.2
                  imagePullPolicy: IfNotPresent
                  ports:
                    - containerPort: 8000
                  resources:
                    requests:
                      cpu: "100m"
                      memory: "128Mi"
                    limits:
                      cpu: "1000m"
                      memory: "512Mi"

```
应用配置:
```shell
$ kubectl apply -f deployment.yaml -n k3s-rust-demo
```

### 6. 创建服务 {id="create-service"}

接着创建 service 配置文件 `service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: k3s-rust-demo-svc
  namespace: k3s-rust-demo
spec:
  type: ClusterIP
  selector:
    app: k3s-rust-demo
  ports:
    - port: 80
      targetPort: 8000
```

应用配置:
```shell
$ kubectl apply -f service.yaml -n k3s-rust-demo
```

### 7. 对外提供服务 {id="create-ingress"}

最后，对外提供 Web 服务，并且管理 SSL 证书:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k3s-rust-demo-ingress
  namespace: k3s-rust-demo
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Traefik 启用 HTTPS 重定向
    traefik.ingress.kubernetes.io/entrypoints: "websecure"
    traefik.ingress.kubernetes.io/redirect-entrypoint: "websecure"
spec:
  ingressClassName: "traefik"
  tls:
    - hosts:
        - rust.demo.testing.icu
      secretName: k3s-rust-demo-tls
  rules:
    - host: rust.demo.testing.icu
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: k3s-rust-demo-svc
                port:
                  number: 80
```
应用配置:
```yaml
$ kubectl apply -f ingress-with-tls.yaml -n k3s-rust-demo
```
完成之后，可以通过域名访问你的服务了。

## GitHub CI {id="github-ci"}

接下来就可以集成 GitHub CI 了，让这一切都自动发生。CI 整个过程如下:

```text
检出代码 -> 安装依赖 -> 缓存依赖 -> 发布镜像 -> 设置 kube 凭证 -> 滚动更新
```

接下来就按照这个过程，分步编写 GitHub CI 的配置文件。你可以先在项目根目录下创建配置文件:`.github/workflows/main.yaml`。

### 1. 获取 Kube 配置 {id="get-kube-config"}

首先要获取 Kube 配置，作为使用在 GitHub CI 服务器上使用 `kubectl` 操作集群的认证凭证。
```shell
# 将配置扁平化、最小化后通过 base64 编码
kubectl config view --raw --flatten --minify | base64 -w0
```

然后注入到 GitHub 仓库的 Secrets 中去，位于仓库的 `Settings->Security->Secrets and variables->Actions` 中:

![08350de4ec208ce80695ef2055e51356.png](https://i.miji.bid/2025/05/05/08350de4ec208ce80695ef2055e51356.png)

之后就可以在 CI 的配置文件中安全地使用这个变量了。

### 2. 构建并发布镜像 {id="build-push-image"}

接着我们就来编写 GitHub CI 的配置了，编辑之前创建`.github/workflows/main.yaml`：
```yaml
name: Deploy to prod

on:
  push:
    branches:
      - main
```
上面的配置是给这次 CI 命名，然后指出当 `main` 分支发生 push 的时候，触发 CI。接下来定义 `jobs`, 首先需要构建和发布镜像，配置如下:
```yaml
jobs:
  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.tag.outputs.IMAGE_TAG }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Determine image tag
        id: tag
        run: |
          # 取前 8 位提交 SHA 作为镜像标签
          echo "IMAGE_TAG=${GITHUB_SHA::8}" >> $GITHUB_OUTPUT

      - name: Log in to Docker registry
        uses: docker/login-action@v2
        with:
          registry: ${{ secrets.DOCKER_REGISTRY }}
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build & push image
        uses: docker/build-push-action@v3
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ${{ secrets.DOCKER_REGISTRY }}/k3s-rust-demo:${{ steps.tag.outputs.IMAGE_TAG }}
```
这个脚本涉及三个 secrets，也需要提前在 GitHub 中配置，分别是:

* `DOCKER_REGISTRY`：私有仓库的域名
* `DOCKER_USERNAME`：登录私有仓库的用户名
* `DOCKER_PASSWORD`：登录私有仓库的密码

利用之前已经编写好了 Dockerfile 分阶段编译，打包后的大小仅 80MB 左右，然后构建镜像发布到私有的仓库中。

### 3. 滚动更新应用 {id="rolling-update"}

最后，使用 `kubectl` 命令，远程滚动更新应用即可，这一步的配置如下：
```yaml
deploy:
    name: Deploy to k3s
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.27.0'

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_DATA }}" | base64 --decode > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      - name: Update Deployment image
        run: |
          kubectl -n k3s-rust-demo set image deployment/k3s-rust-demo \
            k3s-rust-demo="${{ secrets.DOCKER_REGISTRY }}/k3s-rust-demo:${{ needs.build-and-push.outputs.image-tag }}"
          kubectl -n k3s-rust-demo rollout status deployment/k3s-rust-demo
```

## 总结 {id="summary"}

这篇文档介绍了如何在 k3s 上部署 Rust 的 Web 应用。