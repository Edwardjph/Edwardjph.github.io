---
title: Centos安装k8s(学习环境，使用Minikube安装Kubernetes)
tags: k8s
---

---

## 一、环境说明

#### 系统

```shell
cat /etc/redhat-release
CentOS Linux release 7.7.1908 (Core）
```

**注：创建虚拟机时在处理器处勾选：虚拟化Inter VT-x/EPT or AMD-V/RVI**

![image-20200216](\assets\images\k8s\image-20200216.png)

#### 安装docker（[参考Centos安装docker](https://edwardjph.github.io/2020/02/12/docker.html)）

### 二、设置系统代理（[Minikube使用](https://minikube.sigs.k8s.io/docs/reference/networking/proxy/)）

```shell
vi /etc/profile
```

在最后加入：

```shell
export HTTP_PROXY=http://<proxy hostname:port>
export HTTPS_PROXY=https://<proxy hostname:port>
export NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.80.4/24
```

**注：192.168.80.4/24：由minikube VM使用（虚拟机ip），10.96.0.0/12：由服务集群IP使用**

```shell
source /etc/profile
service network restart
```

### 三、安装minikube（[基于官网教程](https://kubernetes.io/docs/setup/learning-environment/minikube/)）

#### 在开始前检查linux是否支持虚拟化

```shell
grep -E --color 'vmx|svm' /proc/cpuinfo
```

#### 安装kubectl

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl
```

#### 下载minikube

```shell
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
```

#### 安装minikube

```shell
mkdir -p /usr/local/bin/
install minikube /usr/local/bin/
```

**注：在你运行之前：**

1. 关闭防火墙：

   ```shell
   systemctl stop firewalld
   systemctl disable firewalld
   ```

2. 关闭swap：
   临时关闭：

   ```shell
   swapoff -a
   ```

   永久关闭：

   ```shell
   vi /etc/fstab
   ```

   将其中swap一行注释掉

3. 关闭SELinux

   ```shell
   setenforce 0
   sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
   ```

4. 修改sysctl内核参数

   ```shell
   cat <<EOF > /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   sysctl --system
```
   
5. 设置docker为开机自启

   ```shell
   systemctl enable docker
   ```



### 四、使用Minikube安装Kubernetes

#### 运行minikube

```shell
minikube start --vm-driver=none
```

**报错：**

- 虚拟机无权访问 k8s.gcr.io：
  系统代理未生效，重新配置代理，多试几次

- /proc/sys/net/ipv4/ip_forward contents are not set to 1

  ```shell
  echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
  ```

- 还有一些警告依次解决，均为运行前注意内容

- 此处docker必须设置代理

运行成功执行：

```shell
minikube status
```

显示：

```shell
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

若未成功，执行：

```shell
minikube delete
```

修改bug！重新运行！

#### 测试

1. 使用名为hello-minikube的映像创建一个Kubernetes部署

   ```shell
   kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.10
   ```

   输出以下内容：

   ```
   deployment.apps/hello-minikube created
   ```

2. 要访问`hello-minikube`部署，请将其公开为服务

   ```shell
   kubectl expose deployment hello-minikube --type=NodePort --port=8080
   ```

   该选项`--type=NodePort`指定服务的类型。

   输出类似于以下内容：

   ```
   service/hello-minikube exposed
   ```

3. 检查Pod是否已启动并正在运行

   ```shell
   kubectl get pod
   ```

   如果输出显示`STATUS`为`ContainerCreating`，则仍在创建Pod：

   ```
   NAME                              READY     STATUS              RESTARTS   AGE
   hello-minikube-3383150820-vctvh   0/1       ContainerCreating   0          3s
   ```

   如果输出显示`STATUS`为`Running`，则Pod现在已启动并正在运行：

   ```
   NAME                              READY     STATUS    RESTARTS   AGE
   hello-minikube-3383150820-vctvh   1/1       Running   0          13s
   ```

4. 获取公开服务的URL以查看服务详细信息：

   ```shell
   minikube service hello-minikube --url
   ```

5. 在浏览器中复制并粘贴作为输出获得的URL
   输出类似于以下内容：

   ```
   Hostname: hello-minikube-797f975945-nqtp5
   
   Pod Information:
   	-no pod information available-
   
   Server values:
   	server_version=nginx: 1.13.3 - lua: 10008
   
   Request Information:
   	client_address=172.17.0.1
   	method=GET
   	real path=/
   	query=
   	request_version=1.1
   	request_scheme=http
   	request_uri=http://192.168.80.4:8080/
   
   Request Headers:
   	accept=text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
   	accept-encoding=gzip, deflate
   	accept-language=zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7,en-US;q=0.6,ko-KR;q=0.5,ko;q=0.4
   	cache-control=max-age=0
   	connection=keep-alive
   	host=192.168.80.4:31655
   	upgrade-insecure-requests=1
   	user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36
   
   Request Body:
   	-no body in request-
   
   ```

6. 删除`hello-minikube`服务：

   ```shell
   kubectl delete services hello-minikube
   ```

   输出类似于以下内容：

   ```
   service "hello-minikube" deleted
   ```

7. 删除`hello-minikube`部署：

   ```shell
   kubectl delete deployment hello-minikube
   ```

   输出类似于以下内容：

   ```
   deployment.extensions "hello-minikube" deleted
   ```

8. 停止本地Minikube集群：

   ```shell
   minikube stop
   ```

   输出类似于以下内容：

   ```
   Stopping "minikube"...
   "minikube" stopped.
   ```