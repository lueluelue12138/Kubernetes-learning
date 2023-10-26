# 这是一个极简快速入门的教程
## 安装[Docker](https://www.docker.com/get-started/)和[Minikube](https://minikube.sigs.k8s.io/docs/start/)
下面给出Mac上的安装命令，其他平台请参考官方文档。
```bash
$ brew install --cask docker
$ brew install minikube
```
## 启动Docker和Minikube
```bash
# 此处默认你已经启动了Docker
$ minikube start
.
.
.
$ minikube status # 可以通过这个命令查看minikube的启动状态
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
$ minikube dashboard # 可以通过这个命令在浏览器中打开本地Kubernetes集群的监控面板(ps:请不要关闭运行这个指令的终端,或者你可以在后台运行这个指令)
🤔  Verifying dashboard health ...
🚀  Launching proxy ...
🤔  Verifying proxy health ...
🎉  Opening http://127.0.0.1:61304/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
$ minikube stop # 停止本地Kubernetes集群
.
.
.
```
## 部署一个应用
我会在后面解释这个配置文件的的含义，现在你只需要知道这个配置文件是用来部署一个应用的就可以了。
```bash
# 使用.yaml文件部署一个应用
$ kubectl apply -f deployment.yaml
deployment.apps/finance created
service/finance-np-service created
$ kubectl get pods # 查看pod的状态
NAME                       READY   STATUS    RESTARTS   AGE
finance-6dfc7584dd-2gs2r   1/1     Running   0          3m20s
finance-6dfc7584dd-qp8n8   1/1     Running   0          3m20s
finance-6dfc7584dd-w5rj2   1/1     Running   0          3m20s
$ kubectl get services # 查看service的状态
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
finance-np-service   NodePort    10.104.160.229   <none>        5000:30080/TCP   3m49s
kubernetes           ClusterIP   10.96.0.1        <none>        443/TCP          16h
$ minikube service finance-np-service # 在浏览器中打开finance-np-service的访问地址
.
.
.
$ kubectl delete -f deployment.yaml # 删除刚才部署的应用
deployment.apps "finance" deleted
service "finance-np-service" deleted
``` 
## 理解deployment.yaml配置文件
Kubernetes集群中部署的最小单位为pod,下面的配置文件指定了pod的模版，也就是说，我们可以利用这个模版部署多个pod。这里的port指定了应用的端口号，这个端口号是pod内部的端口号，也就是说，我们可以通过pod的IP地址和这个端口号访问应用。但是，pod的IP地址是动态分配的，我们无法预先知道，所以，我们需要一个service来代理pod。
```yaml
template:
    metadata:
      labels:
        app: finance
    spec:
      containers:
      - name: finance
        image: rossning92/finance
        resources:
          limits:
            memory: "256Mi" # 这里指定了pod的内存使用量为256MiB
            cpu: "500m" # "500m"表示pod使用0.5个CPU核心
        ports:
        - containerPort: 5000 # 这里指定了pod的端口号为5000
```
service掌握了pod的IP地址，并且可以通过这个IP地址和端口号访问pod应用。但是，service的IP地址也是动态分配的，我们无法预先知道，同时为了方便外部访问，我们需要一个nodePort来代理这个service。(这里port即是service的端口号，targetPort即是pod的端口号)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: finance-np-service
spec:
  selector:
    app: finance
  ports:
  - port: 5000 # 这里指定了service的端口号为5000
    targetPort: 5000 # 这里指定了service代理的pod的端口号为5000
```
通过外部IP和nodePort可以访问nodePort代理的service，继而通过这个service访问pod应用。这里的nodePort是30080。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: finance-np-service
spec:
  selector:
    app: finance
  type: NodePort # 这里指定了service的类型为NodePort
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30080 # 这里指定了nodePort的端口号为30080
```
这里的replicas指定了pod的数量为3，也就是说，我们会部署3个pod。这3个pod会被service代理，service会被nodePort代理。需要注意的是当外部请求通过nodePort（在这个例子中是30080）到达本地Kubernetes节点时，Service负责将这些请求路由到标签为app: finance的Pod中。由于有3个这样的Pod，请求会被负载均衡地转发到这三个Pod中的任意一个。
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: finance
spec:
  replicas: 3
  selector:
    matchLabels:
      app: finance
```
## 参考资料
[[1] Kubernetes (k8s) 10分钟快速入门](https://www.bilibili.com/video/BV1DL4y187cL)  
[[2] Minikube Official Documentation](https://minikube.sigs.k8s.io/docs/)  
[[3] Kubernetes Official Documentation](https://kubernetes.io/docs/home/)