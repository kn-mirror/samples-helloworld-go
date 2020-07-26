用`Go`写的一个简单的 Web 应用程序 您可以用于测试.
它读取的环境变量`TARGET`并打印`Hello ${TARGET}!`.
如果 `TARGET` 未指定, 它将使用`World`作为`TARGET`.

按照下面创建示例代码，然后部署应用到集群的步骤。
您也可以下载样本的工作副本，通过运行以下命令:

```shell
git clone -b "{{< branch >}}" https://github.com/knative/docs knative-docs
cd knative-docs/docs/serving/samples/hello-world/helloworld-go
```

## 在你开始之前

- 带 Knative 安装和配置 DNS 的 Kubernetes 集群 .
  如果你需要创建一个请按照[安装说明](../../../../install/README.md) .
- [Docker](https://www.docker.com) 安装在本地计算机上运行, 和配置 Docker Hub 账户 (我们将使用它的容器注册表).

## 重新创建示例代码

1. 创建的文件名为`helloworld.go` 并粘贴以下代码.此代码创建监听 8080 端口上一个基本的 Web 服务器:

   ```go
   package main

   import (
     "fmt"
     "log"
     "net/http"
     "os"
   )

   func handler(w http.ResponseWriter, r *http.Request) {
     log.Print("helloworld: received a request")
     target := os.Getenv("TARGET")
     if target == "" {
       target = "World"
     }
     fmt.Fprintf(w, "Hello %s!\n", target)
   }

   func main() {
     log.Print("helloworld: starting server...")

     http.HandleFunc("/", handler)

     port := os.Getenv("PORT")
     if port == "" {
       port = "8080"
     }

     log.Printf("helloworld: listening on port %s", port)
     log.Fatal(http.ListenAndServe(fmt.Sprintf(":%s", port), nil))
   }
   ```

2. 在你的项目目录, 创建一个名为`Dockerfile`文件和复制下面的代码块到它. 有关 dockerizing 一个`Go`应用程序详细的说明, 参见[用`Docker`部署`Go`服务器](https://blog.golang.org/docker).

   ```docker
   # Use the official Golang image to create a build artifact.
   # This is based on Debian and sets the GOPATH to /go.
   # https://hub.docker.com/_/golang
   FROM golang:1.13 as builder

   # Create and change to the app directory.
   WORKDIR /app

   # Retrieve application dependencies using go modules.
   # Allows container builds to reuse downloaded dependencies.
   COPY go.* ./
   RUN go mod download

   # Copy local code to the container image.
   COPY . ./

   # Build the binary.
   # -mod=readonly ensures immutable go.mod and go.sum in container builds.
   RUN CGO_ENABLED=0 GOOS=linux go build -mod=readonly -v -o server

   # Use the official Alpine image for a lean production container.
   # https://hub.docker.com/_/alpine
   # https://docs.docker.com/develop/develop-images/multistage-build/#use-multi-stage-builds
   FROM alpine:3
   RUN apk add --no-cache ca-certificates

   # Copy the binary to the production image from the builder stage.
   COPY --from=builder /app/server /server

   # Run the web service on container startup.
   CMD ["/server"]
   ```

3. 创建一个新文件, `service.yaml` 和下面的服务定义复制到该文件.
   请确保您的 Docker Hub 用户名来替换`{username}`。

   ```yaml
   apiVersion: serving.knative.dev/v1
   kind: Service
   metadata:
     name: helloworld-go
     namespace: default
   spec:
     template:
       spec:
         containers:
           - image: docker.io/{username}/helloworld-go
             env:
               - name: TARGET
                 value: "Go Sample v1"
   ```

4. 使用`go`工具来创建[`go.mod`](https://github.com/golang/go/wiki/Modules#gomod)清单.

   ```shell
   go mod init github.com/knative/docs/docs/serving/samples/hello-world/helloworld-go
   ```

## 构建和部署样本

一旦你重新创建示例代码文件(或用在样品夹中的文件)，您已经准备好构建和部署示例应用程序.

1. 使用`Docker`打造的示例代码放入容器.
   使用 Docker Hub 建立和推 , 运行这些命令使用你的 Docker Hub 用户名替换了`{username}`:

   ```shell
   # Build the container on your local machine
   docker build -t {username}/helloworld-go .

   # Push the container to docker registry
   docker push {username}/helloworld-go
   ```

2. 之后构建已经完成，并且将容器压到 docker hub, 您可以部署应用程序到您的集群.
   确保在`service.yaml`容器镜像值 匹配的容器，你在上一步中建.
   使用`kubectl`应用配置:

   ```shell
   kubectl apply --filename service.yaml
   ```

3. 现在已经创建了您的服务, Knative 将执行以下步骤:

   - 此版本的应用程序创建一个新的不可改变的版本。
   - 网络编程创建一个路由，入口，服务和负载您的应用程序的平衡。
   - 自动调整你的`pods`上下 (包括零活性`pods`).

4. 运行以下命令来寻找您的服务域 URL:

   ```shell
   kubectl get ksvc helloworld-go  --output=custom-columns=NAME:.metadata.name,URL:.status.url
   ```

   例:

   ```shell
    NAME                URL
    helloworld-go       http://helloworld-go.default.1.2.3.4.xip.io
   ```

5. 现在你可以对您的应用程序的请求并查看结果.
   使用上一个命令返回的 URL 替换下面的网址.

   ```shell
   curl http://helloworld-go.default.1.2.3.4.xip.io
   Hello Go Sample v1!
   ```

   > 注意: 添加`-v`选项，以获得更多的细节，如果`curl`命令失败.

## 删除示例应用程序部署

若要从群集中删除示例应用程序, 删除服务记录:

```shell
kubectl delete --filename service.yaml
```
