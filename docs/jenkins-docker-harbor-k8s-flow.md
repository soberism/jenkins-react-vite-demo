# Jenkins + Docker + Harbor + Kubernetes 前端部署流程总结

这份文档记录我们这次从 0 到 1 搭起来的前端 CI/CD 流程。

项目是一个 React + Vite 前端项目，最终目标是：

```text
GitHub 代码
  ↓
Jenkins 自动拉代码
  ↓
npm ci / npm run build
  ↓
生成 dist/
  ↓
Docker 构建 Nginx 镜像
  ↓
推送镜像到 Harbor
  ↓
Kubernetes 拉取镜像并部署
  ↓
浏览器访问 http://localhost
```

## 1. Git 仓库做了什么

Git 仓库负责保存项目代码和部署配置。

我们创建了一个 React + Vite 项目，并推送到 GitHub：

```text
https://github.com/soberism/jenkins-react-vite-demo
```

仓库里的关键文件：

```text
Jenkinsfile
Dockerfile
nginx.conf
k8s/deployment.yaml
k8s/service.yaml
```

这些文件分别负责：

```text
Jenkinsfile          Jenkins 自动化流水线
Dockerfile           构建前端 Docker 镜像
nginx.conf           Nginx 静态资源服务配置
k8s/deployment.yaml  Kubernetes 应用部署配置
k8s/service.yaml     Kubernetes 应用访问入口配置
```

所以这个仓库不只是保存前端代码，也保存了构建、镜像、部署相关配置。

## 2. Jenkins 做了什么

Jenkins 是自动化工具，也可以理解成整个流程的指挥官。

它负责把原本需要手动做的事情串起来：

```text
拉代码
安装依赖
前端打包
构建 Docker 镜像
推送 Harbor
部署 Kubernetes
```

我们的 `Jenkinsfile` 大致分成这些阶段：

```text
Checkout        拉取 GitHub 代码
Install         执行 npm ci
Build           执行 npm run build
Archive         保存 dist 构建产物
Docker Build    构建 Docker 镜像
Docker Push     推送镜像到 Harbor
Deploy to k8s   部署到 Kubernetes
```

Jenkins 里点击一次 `Build Now`，就会自动执行上面整套流程。

这就是 CI/CD 的核心：

```text
CI: 持续集成，自动拉代码、安装依赖、构建、检查
CD: 持续部署，自动构建镜像、推仓库、部署到服务器或集群
```

## 3. Docker 做了什么

Docker 负责把项目变成一个可以到处运行的镜像。

前端项目执行：

```bash
npm run build
```

之后只会生成：

```text
dist/
```

但是 `dist/` 只是静态文件，它自己不能运行。

所以我们用 Dockerfile 把 `dist/` 放进 Nginx 镜像里：

```text
dist/ + nginx = 一个可运行的前端 Docker 镜像
```

我们的 Dockerfile 做了三件事：

```text
1. 基于 nginx:1.27-alpine
2. 复制 nginx.conf 到 Nginx 配置目录
3. 复制 dist 到 /usr/share/nginx/html
```

构建出来的镜像类似：

```text
localhost:8089/demo/jenkins-react-vite-demo:5
localhost:8089/demo/jenkins-react-vite-demo:latest
```

其中：

```text
5       Jenkins 第 5 次构建生成的版本
latest  最新版本的别名
```

真实项目里更推荐用构建号、Git commit、版本号这种明确 tag，而不是只依赖 `latest`。

## 4. Nginx 做了什么

Nginx 是容器里真正提供网页访问的服务。

React + Vite 打包后的文件在：

```text
dist/
```

Docker 构建镜像时，会把这些文件复制到 Nginx 的网页目录：

```text
/usr/share/nginx/html
```

浏览器访问应用时，其实访问的是容器里的 Nginx。

我们的 `nginx.conf` 主要做两件事：

```text
1. 访问 / 时返回 index.html
2. 支持前端路由刷新，不会直接 404
```

关键配置是：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

这行对 React 单页应用很重要。

比如以后有前端路由：

```text
/login
/dashboard
/user/1
```

刷新页面时，Nginx 找不到真实文件，就会回退到 `index.html`，再交给 React 自己处理路由。

## 5. Harbor 做了什么

Harbor 是镜像仓库。

可以把它理解成公司内部的 Docker Hub。

Jenkins 构建完镜像后，需要把镜像放到一个公共位置，让 Kubernetes 能拉到。

这个公共位置就是 Harbor。

我们本地 Harbor 地址是：

```text
http://localhost:8089
```

项目名是：

```text
demo
```

镜像仓库是：

```text
jenkins-react-vite-demo
```

完整镜像地址类似：

```text
localhost:8089/demo/jenkins-react-vite-demo:5
```

Harbor 页面主要用来看：

```text
镜像有没有推送成功
有哪些 tag
每个 tag 是什么时候推送的
镜像大小是多少
```

Harbor 不负责运行应用，它只负责存镜像。

## 6. Kubernetes 做了什么

Kubernetes 负责运行和管理镜像。

Docker 可以直接运行容器：

```bash
docker run ...
```

但是真实公司环境里，通常不会直接靠一个 `docker run` 管线上服务。

Kubernetes 会负责：

```text
运行容器
自动重启
多副本
滚动发布
服务发现
统一暴露访问入口
```

我们使用的是 Docker Desktop 自带的本地 Kubernetes。

当前集群大概是：

```text
Cluster type: kind
Nodes: 1
Kubernetes version: v1.36.1
```

这是一个本地学习用的单节点集群。

## 7. k8s/deployment.yaml 做了什么

`deployment.yaml` 负责说明应用怎么运行。

核心内容包括：

```text
应用名称
副本数量
使用哪个镜像
容器暴露哪个端口
```

当前配置里，镜像是：

```text
localhost:8089/demo/jenkins-react-vite-demo:v1
```

但是 Jenkins 真正发布时，会执行：

```bash
kubectl set image deployment/jenkins-react-vite-demo web=localhost:8089/demo/jenkins-react-vite-demo:${BUILD_NUMBER}
```

所以最终运行的镜像 tag 会变成 Jenkins 构建号，例如：

```text
localhost:8089/demo/jenkins-react-vite-demo:5
```

Deployment 还会带来几个能力：

```text
Pod 挂了自动重启
修改镜像后滚动发布
可以把 replicas 改成多个副本
可以回滚到旧版本
```

## 8. k8s/service.yaml 做了什么

`service.yaml` 负责把应用暴露出来。

Deployment 只负责运行 Pod，但是 Pod 的地址会变化，不能直接依赖。

Service 的作用是给 Pod 提供一个稳定入口。

我们的 Service 类型是：

```text
LoadBalancer
```

本地 Docker Desktop 会把它映射到：

```text
http://localhost
```

所以浏览器访问：

```text
http://localhost
```

最终会访问到 Kubernetes 里面运行的前端 Pod。

## 9. Docker 直接部署和 k8s 部署的区别

一开始我们用 Docker 直接部署：

```text
npm run build
  ↓
docker build
  ↓
docker run
  ↓
访问 http://localhost:18080
```

这个流程简单，适合本地验证。

后来我们改成 Kubernetes 部署：

```text
npm run build
  ↓
docker build
  ↓
docker push 到 Harbor
  ↓
k8s 从 Harbor 拉镜像
  ↓
k8s 创建 Deployment / Pod / Service
  ↓
访问 http://localhost
```

区别是：

```text
Docker 直接部署：自己手动运行一个容器
k8s 部署：交给集群管理容器
```

k8s 更接近真实公司部署方式。

## 10. 每个工具之间的关系

可以这样理解：

```text
GitHub
  保存代码和配置

Jenkins
  自动执行构建和部署流程

Docker
  把代码打包成镜像

Nginx
  在镜像里提供前端静态网页服务

Harbor
  保存 Docker 镜像

Kubernetes
  从 Harbor 拉镜像，并运行、管理应用
```

完整关系图：

```text
开发者 push 代码到 GitHub
        ↓
Jenkins 拉取 GitHub 代码
        ↓
Jenkins 执行 npm ci / npm run build
        ↓
Jenkins 调用 Docker 构建镜像
        ↓
Jenkins 把镜像推送到 Harbor
        ↓
Jenkins 调用 kubectl 更新 k8s Deployment
        ↓
k8s 从 Harbor 拉取新镜像
        ↓
k8s 滚动更新 Pod
        ↓
用户访问 Service 暴露出来的地址
```

## 11. 怎么看每个环节是否成功

看 Jenkins：

```text
所有 Stage 都是绿色
Docker Build 成功
Docker Push 成功
Deploy to k8s 成功
```

看 Harbor：

```text
进入 demo 项目
查看 jenkins-react-vite-demo 仓库
确认有新的 tag
```

看 Kubernetes：

```bash
kubectl get deployment
kubectl get pods
kubectl get service
```

确认类似：

```text
deployment/jenkins-react-vite-demo   Available
pod/jenkins-react-vite-demo-xxx      Running
service/jenkins-react-vite-demo      80/TCP
```

看当前 k8s 运行的是哪个镜像：

```bash
kubectl get deployment jenkins-react-vite-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
```

看页面：

```text
http://localhost
```

## 12. 现在这套流程的状态

目前我们已经完成：

```text
React + Vite 项目创建
GitHub 仓库推送
Jenkins Pipeline 创建
NodeJS 工具配置
Docker 镜像构建
Nginx 静态服务配置
Harbor 本地镜像仓库启动
Harbor demo 项目创建
镜像推送到 Harbor
Docker Desktop Kubernetes 启动
k8s Deployment / Service 创建
Jenkins 自动部署到 k8s
浏览器访问 http://localhost 成功
```

这已经是一套完整的本地 CI/CD 学习环境。

## 13. 后续可以继续学习什么

下一步可以继续补这些内容：

```text
1. 把 replicas 从 1 改成 2 或 3，学习多副本
2. 修改页面内容，重新构建，观察滚动发布
3. 用 kubectl rollout history 查看发布历史
4. 用 kubectl rollout undo 学习回滚
5. 给 k8s 加 readinessProbe / livenessProbe
6. 给应用加 Ingress，用域名访问
7. 把 Harbor 从 localhost 换成真实 IP 或域名
8. 给 Harbor 配 HTTPS
9. 把 Jenkins 构建触发方式改成 GitHub webhook
```

## 14. 一句话总结

```text
Jenkins 负责自动化流程，
Docker 负责打包镜像，
Nginx 负责提供前端页面，
Harbor 负责保存镜像，
Kubernetes 负责运行和管理镜像，
GitHub 负责保存代码和配置。
```

