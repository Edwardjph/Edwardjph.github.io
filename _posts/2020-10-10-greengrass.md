# 在 Docker 容器中运行 AWS IoT Greengrass

环境：Ubuntu 18.04.1

用户：user

连接器在docker容器中运行，无法访问外部

**注：安装python3.7时，安装[openssl](https://blog.51cto.com/13544424/2149473)！！！！**

### 安装docker

```
curl -sSL https://get.daocloud.io/docker | sh
```

### 安装 AWS CLI 版本 2

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

配置CLI2

```bash
$ aws configure
AWS Access Key ID [None]: xxxxxxxxxxxxxxx
AWS Secret Access Key [None]: xxxxxxxxxxxxxxxxx
Default region name [None]: us-east-2
Default output format [None]: json
```

### 从 Amazon ECR 获取 AWS IoT Greengrass 容器镜像

登录到 Amazon ECR 中的 AWS IoT Greengrass 镜像仓库

```
aws ecr get-login-password --region  us-west-2 | docker login --username AWS --password-stdin https://216483018798.dkr.ecr.us-west-2.amazonaws.com
```

检索 AWS IoT Greengrass 容器镜像

```
docker pull 216483018798.dkr.ecr.us-west-2.amazonaws.com/aws-iot-greengrass:latest
```

启用符号链接和硬链接保护

```
echo 1 > /proc/sys/fs/protected_hardlinks
echo 1 > /proc/sys/fs/protected_symlinks
#保证重启后保持不变
echo '# AWS Greengrass' >> /etc/sysctl.conf 
echo 'fs.protected_hardlinks = 1' >> /etc/sysctl.conf 
echo 'fs.protected_symlinks = 1' >> /etc/sysctl.conf

sysctl -p
```

启用 IPv4 网络转发

```
sudo vim /etc/sysctl.conf
# set this net.ipv4.ip_forward = 1
sudo sysctl -p
```

### [创建和配置 Greengrass 组和核心](https://docs.aws.amazon.com/zh_cn/greengrass/latest/developerguide/gg-config.html)

### 配置网络代理

停止core

```
cd /greengrass-root/ggc/core/
sudo ./greengrassd stop
```

修改配置

```
{
    "coreThing" : {
        "caPath" : "root.ca.pem",
        "certPath" : "12345abcde.cert.pem",
        "keyPath" : "12345abcde.private.key",
        "thingArn" : "arn:aws:iot:us-west-2:123456789012:thing/core-thing-name",
        "iotHost" : "abcd123456wxyz-ats.iot.us-west-2.amazonaws.com",
        "ggHost" : "greengrass-ats.iot.us-west-2.amazonaws.com",
        "keepAlive" : 600,
        "networkProxy": {
            "noProxyAddresses" : "http://128.12.34.56,www.mywebsite.com",
            "proxy" : {
                "url" : "https://my-proxy-server:1100",
                "username" : "Mary_Major",
                "password" : "pass@word1357"
            }
        }
    },
    ...
}
```

启动core

```
cd /greengrass-root/ggc/core/
sudo ./greengrassd start
```

### 本地运行 AWS IoT Greengrass

将证书和配置文件（在创建 Greengrass 组时下载的）解压到一个已知位置

```
tar xvzf hash-setup.tar.gz -C /tmp/
```

对于 ATS 端点（首选），请下载恰当的 ATS 根 CA 证书

```
cd /tmp/certs/
sudo wget -O root.ca.pem https://www.amazontrust.com/repository/AmazonRootCA1.pem
```

启动 AWS IoT Greengrass 并将证书和配置文件绑定挂载到 Docker 容器中

```
docker run --rm --init -it --name aws-iot-greengrass \
--entrypoint /greengrass-entrypoint.sh \
-v /tmp/certs:/greengrass/certs \
-v /tmp/config:/greengrass/config \
-p 8883:8883 \
216483018798.dkr.ecr.us-west-2.amazonaws.com/aws-iot-greengrass:latest
```

### 部署运行时日志

`/greengrass/ggc/var/log/system/runtime.log`

### 为Greengrass组配置“无容器”容器化

在 Docker 容器中运行 AWS IoT Greengrass 时，所有 Lambda 函数都必须在不进行容器化的情况下运行

1. 在 AWS IoT 控制台中，选择 **Greengrass**，然后选择 **Groups (组)**。

2. 

   选择要更改其设置的组。

3. 

   选择 **Settings**。

4. 在 **Lambda runtime environment (Lambda 运行时环境)** 下，选择 **No container (无容器)**。

5. 选择 **Update default Lambda execution configuration (更新默认 Lambda 执行配置)**。查看确认窗口中的消息，然后选择 **Continue (继续)**。

### 配置日志

您的 Greengrass 组角色必须允许 AWS IoT Greengrass 写入 CloudWatch Logs

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:DescribeLogStreams"
            ],
            "Resource": [
                "arn:aws:logs:*:*:*"
            ]
        }
    ]
}
```

### 已root身份运行

编辑 config.json 文件以向 `runtime` 字段添加 `"allowFunctionsToRunAsRoot" : "yes"`

```
{
  "coreThing" : {
    ...
  },
  "runtime" : {
    ...
    "allowFunctionsToRunAsRoot" : "yes"
  },
  ...
}
```

### Docker 应用程序部署连接器

**前置条件：**

- AWS IoT Greengrass Core 软件 v1.10 or later。
- [Python](https://www.python.org/) 版本 3.7 已安装在核心设备上并且已添加到 PATH 环境变量。
- Greengrass 内核上至少有 36 MB RAM
- Greengrass 核心上安装的 [Docker Engine](https://docs.docker.com/install/) 1.9.1 或更高版本。`docker` 可执行文件必须位于 `/usr/bin` 或 `/usr/local/bin` 目录中
- Greengrass 核心上安装的 [Docker Compose](https://docs.docker.com/compose/install/)。`docker-compose` 可执行文件必须位于 `/usr/bin` 或 `/usr/local/bin` 目录中。
- [Greengrass 组角色](https://docs.aws.amazon.com/zh_cn/greengrass/latest/developerguide/group-role.html)，配置为允许对包含 Compose 文件的 S3 存储桶执行 `s3:GetObject` 操作
- 如果您的 Docker Compose 文件引用 Amazon ECR 中存储的 Docker 映像，则为配置为允许执行以下操作的 [Greengrass 组角色](https://docs.aws.amazon.com/zh_cn/greengrass/latest/developerguide/group-role.html)：
  - 对包含 Docker 镜像的 Amazon ECR 存储库执行的 `ecr:GetDownloadUrlForLayer` 和 `ecr:BatchGetImage` 操作。
  - 对您的资源执行的 `ecr:GetAuthorizationToken` 操作
- 如果您的 Docker Compose 文件引用来自 [AWS Marketplace](https://aws.amazon.com/marketplace) 的 Docker 镜像，则连接器还具有以下要求：
  - 您必须订阅 AWS Marketplace 容器产品。
  - AWS IoT Greengrass 必须配置为支持本地密钥，如[密钥要求](https://docs.aws.amazon.com/zh_cn/greengrass/latest/developerguide/secrets.html#secrets-reqs)中所述。 连接器仅使用此功能从 AWS Secrets Manager 中检索您的密钥，而不存储它们。
  - 您必须在 Secrets Manager 中为用于存储在 Compose 文件中引用的 Docker 镜像的每个 AWS Marketplace 镜像仓库创建一个密钥
- 如果您的 Docker Compose 文件从 Amazon ECR（如 Docker Hub）之外的镜像仓库中的私有存储库引用 Docker 镜像，则连接器还具有以下要求：
  - AWS IoT Greengrass 必须配置为支持本地密钥，如[密钥要求](https://docs.aws.amazon.com/zh_cn/greengrass/latest/developerguide/secrets.html#secrets-reqs)中所述。 连接器仅使用此功能从 AWS Secrets Manager 中检索您的密钥，而不存储它们。
  - 您必须在 Secrets Manager 中为用于存储在 Compose 文件中引用的 Docker 镜像的每个私有存储库创建一个密钥

```
aws greengrass create-connector-definition --name MyGreengrassConnectors --initial-version '{
    "Connectors": [
        {
            "Id": "MyDockerAppplicationDeploymentConnector",
            "ConnectorArn": "arn:aws:greengrass:us-east-2::/connectors/DockerApplicationDeployment/versions/5",
            "Parameters": {
                "DockerComposeFileS3Bucket": "sagemaker-us-east-2-934311906357",
                "DockerComposeFileS3Key": "trainingPlatform/docker-compose.yml",
                "DockerComposeFileDestinationPath": "/home/user/Downloads",
                "DockerUserId": "1000",
                "DockerContainerStatusLogFrequency": "30",
                "ForceDeploy": "True",
                "DockerPullBeforeUp": "True"
            }
        }
    ]
}'
```



# 直接运行AWS IoT Greengrass core

### 安装python3.7

**注：安装python3.7时，安装[openssl](https://blog.51cto.com/13544424/2149473)！！！！**

### 新建用户与组

```
sudo adduser --system ggc_user
sudo groupadd --system ggc_group
```

### 运行以下命令以检查是否启用了硬链接和软链接保护

```
sudo sysctl -a | grep fs.protected
```

启用符号链接和硬链接保护

```
echo 1 > /proc/sys/fs/protected_hardlinks
echo 1 > /proc/sys/fs/protected_symlinks
```

要使设置在重新启动后保持不变

```
echo '# AWS Greengrass' >> /etc/sysctl.conf 
echo 'fs.protected_hardlinks = 1' >> /etc/sysctl.conf 
echo 'fs.protected_symlinks = 1' >> /etc/sysctl.conf

sysctl -p
```

### 提取并运行以下脚本以挂载 [Linux 控制组](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/ch01) (cgroups)。

```
curl https://raw.githubusercontent.com/tianon/cgroupfs-mount/951c38ee8d802330454bdede20d85ec1c0f8d312/cgroupfs-mount > cgroupfs-mount.sh
chmod +x cgroupfs-mount.sh 
sudo bash ./cgroupfs-mount.sh
```

### 运行 Greengrass 依赖项检查程序。

```
mkdir greengrass-dependency-checker-GGCv1.10.x
cd greengrass-dependency-checker-GGCv1.10.x
wget https://github.com/aws-samples/aws-greengrass-samples/raw/master/greengrass-dependency-checker-GGCv1.10.x.zip
unzip greengrass-dependency-checker-GGCv1.10.x.zip
cd greengrass-dependency-checker-GGCv1.10.x
sudo ./check_ggc_dependencies | more
```

### 下载核心软件以及证书到本机

### 解安装 AWS IoT Greengrass Core 软件和安全资源

- 第一条命令在核心设备的根文件夹中创建 `/greengrass` 目录（通过 `-C /` 参数）。
- 第二条命令将核心设备证书和密钥复制到 `/greengrass/certs` 文件夹中，并将 [config.json](https://docs.aws.amazon.com/zh_cn/greengrass/latest/developerguide/gg-core.html#config-json) 文件复制到 `/greengrass/config` 文件夹中（通过 `-C /greengrass` 参数）。

```
sudo tar -xzvf greengrass-OS-architecture-1.10.2.tar.gz -C /
sudo tar -xzvf hash-setup.tar.gz -C /greengrass
cd /greengrass/certs/
sudo wget -O root.ca.pem https://www.amazontrust.com/repository/AmazonRootCA1.pem
```

### 配置core代理

停止core

```
cd /greengrass-root/ggc/core/
sudo ./greengrassd stop
```

修改配置

```
{
    "coreThing" : {
        "caPath" : "root.ca.pem",
        "certPath" : "12345abcde.cert.pem",
        "keyPath" : "12345abcde.private.key",
        "thingArn" : "arn:aws:iot:us-west-2:123456789012:thing/core-thing-name",
        "iotHost" : "abcd123456wxyz-ats.iot.us-west-2.amazonaws.com",
        "ggHost" : "greengrass-ats.iot.us-west-2.amazonaws.com",
        "keepAlive" : 600,
        "networkProxy": {
            "noProxyAddresses" : "http://128.12.34.56,www.mywebsite.com",
            "proxy" : {
                "url" : "https://my-proxy-server:1100",
                "username" : "Mary_Major",
                "password" : "pass@word1357"
            }
        }
    },
    ...
}
```

启动core

```
cd /greengrass-root/ggc/core/
sudo ./greengrassd start
```

### 在核心设备上启动 AWS IoT Greengrass

```
cd /greengrass/ggc/core/
sudo ./greengrassd start
```

### 查看日志

```
vim /greengrass/ggc/var/log/system/runtime.log
```

