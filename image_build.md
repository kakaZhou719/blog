## 背景介绍

## cloud image rootfs 镜像格式介绍

### rootfs 树状图

```
.
├── bin
│   ├── conntrack
│   ├── containerd-rootless-setuptool.sh
│   ├── containerd-rootless.sh
│   ├── crictl
│   ├── kubeadm
│   ├── kubectl
│   ├── kubelet
│   ├── nerdctl
│   └── seautil
├── cri
│   └── docker.tar.gz # cri bin files include docker,containerd,runc. 
├── etc
│   ├── 10-kubeadm.conf
│   ├── Clusterfile  # image default Clusterfile
│   ├── daemon.json # docker daemon config file. 
│   ├── docker.service 
│   ├── kubeadm.yml # kubeadm config including Cluster Configuration,JoinConfiguration and so on.
│   ├── kubelet.service
│   ├── registry_config.yml # docker registry config including storage root directory and http related config.
│   └── registry.yml # If the user wants to customize the username and password, can overwrite this file.
├── images
│   └── registry.tar  # registry docker image, will load this image and run a local registry in cluster
├── Kubefile
├── Metadata
├── README.md
├── registry # will mount this dir to local registry
│   └── docker
│       └── registry
├── scripts
│   ├── clean.sh
│   ├── docker.sh
│   ├── init-kube.sh
│   ├── init-registry.sh
│   ├── init.sh
│   └── kubelet-pre-start.sh
└── statics # yaml files, sealer will render values in those files
    └── audit-policy.yml
```

### 特殊目录介绍

#### images 目录

cloud image docker 镜像文件目录，启动cloud image的时候会将改目录内的离线镜像导入到内置registry中。

样例：复制离线镜像到此目录。

`COPY mysql.tar images`

#### plugins 目录

cloud image 插件文件目录，启动cloud image的时候会将改目录内插件配置加载到runtime中被执行。

样例：复制插件配置文件到此目录。

插件配置 shell.yaml:

```
apiVersion: sealer.aliyun.com/v1alpha1
kind: Plugin
metadata:
  name: taint
spec:
  type: SHELL
  action: PostInstall
  on: node-role.kubernetes.io/master=
  data: |
     kubectl taint nodes --all node-role.kubernetes.io/master-
```

`COPY shell.yaml plugins`

#### charts 目录

cloud image charts文件目录，sealer build 的时候会解析该目录中的charts文件，将对应的容器镜像下载并保存。

样例：nginx charts包到此目录。

`COPY nginx charts`

#### manifests 目录

cloud image manifests文件目录，sealer build 的时候会解析该目录中的yaml文件和imageList文件，将对应的容器镜像下载并保存。

样例：复制imageList文件到manifests目录。

`COPY imageList manifests`

样例：复制dashboard yaml 文件到manifests目录。

`COPY recommend.yaml manifests`

## build 模式介绍

### lite 模式

sealer 默认的构建模式，该种构建模式根据Kubefile的定义，实现不用拉起kubernetes集群的轻量化构建。用于已知镜像清单，或者没有特殊的资源需求的场景。

核心功能介绍

1. 基础集群镜像的拉取和挂载。
2. 根据kubefile执行构建指令,收集相关产物。
3. 根据特殊目录中的构建产物，收集构建过程中产生的容器镜像。
4. 设置cloud image的相关属性并保存镜像。

### local 模式

实现对cloud build模式的本地化构建，以及对无需收集docker镜像的构建方式的支持，例如from scratch的构建。

核心功能介绍

1. 基础集群镜像的拉取和挂载。
2. 根据kubefile执行构建指令,收集相关产物。
3. 根据特殊目录中的构建产物，收集构建过程中产生的容器镜像。
4. 收集对应的集群中pod的镜像。例如启动的operator中容器镜像。
5. 设置cloud image的相关属性并保存镜像。

### cloud 模式

基于云服务，自动化创建ecs并部署kubernetes集群并构建镜像，如果您要交付的环境涉及例如分布式存储这样的底层资源，或者有operator方式启动的工作负载，建议使用此方式来进行构建。

核心功能介绍

1. 使用AKSK,启动云服务的基础资源,ECS,VPC等等。
2. 使用ssh发送构建上下文以及sealer二进制，用于构建环境的准备。
3. 远程执行ssh 命令，完成集群镜像的构建。
4. 清理环境，云服务资源的回收。

### container 模式

与cloud build 原理类似，通过启动多个docker容器作为kubernetes节点（模拟cloud模式的ECS）,从而启动一个kubernetes集群的方式来进行构建。

## build 指令介绍

### FROM 指令

FROM: 引用一个基础镜像，并且Kubefile中第一条指令必须是FROM指令。若基础镜像为私有仓库镜像，则需要仓库认证信息，另外 sealer社区提供了官方的基础镜像可供使用。

> 命令格式：FROM {your base image name}

使用样例：

例如上面示例中,使用sealer 社区提供的`kubernetes:v1.19.8`作为基础镜像。

`FROM registry.cn-qingdao.aliyuncs.com/sealer-io/kubernetes:v1.19.8`

### COPY 指令

COPY: 复制构建上下文中的文件或者目录到rootfs中。

集群镜像文件结构均基于[rootfs结构](../../docs/api/cloudrootfs.md),默认的目标路径即为rootfs，且当指定的目标目录不存在时会自动创建。

如需要复制系统命令，例如复制二进制文件到操作系统的$PATH，则需要复制到rootfs中的bin目录，该二进制文件会在镜像构建和启动时，自动加载到系统$PATH中。

> 命令格式：COPY {src dest}

使用样例：

复制mysql.yaml到rootfs目录中。

`COPY mysql.yaml .`

复制可执行文件到系统$PATH中。

`COPY helm ./bin`

复制远程文件或者git仓库到cloud image中。

`COPY https://github.com/alibaba/sealer/raw/main/applications/cassandra/cassandra-manifest.yaml manifests`

支持通配符复制，将test目录下所有yaml文件复制到manifests中。

`COPY test/*.yaml manifests`

### ARG 指令

ARG: 支持在build阶段设置命令行参数，用于配合CMD和RUN指令使用。

> 命令格式：ARG <参数名>[=<默认值>]

使用样例：

```shell
FROM kubernetes:v1.19.8
# set default version is 4.0.0, this will be used to install mongo application.
ARG Version=4.0.0 
# mongo dir contains many mongo version yaml file.
COPY mongo manifests
# arg Version can be used with RUN instruction.
RUN echo ${Version} 
# use Version arg to install mongo application.
CMD kubectl apply -f mongo-${Version}.yaml 
```

这样就会在build时候启动一个"Version=4.0.0"的yaml.

### RUN 指令

RUN: 使用系统shell执行构建命令，仅在build时运行，可接受多个命令参数，且构建时会将命令执行产物结果保存在镜像中。若系统命令不存在则会构建失败,则需要提前执行COPY指令，将命令复制到镜像中。

> 命令格式：RUN {command args ...}

使用样例：

例如上面示例中,使用wget 命令下载一个kubernetes的dashboard。

`RUN wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml`

### CMD 指令

CMD: 与RUN指令格式类似，使用系统shell执行构建命令。但CMD指令会在镜像启动的时候执行，一般用于启动和配置集群使用。另外与Dockerfile中CMD指令不同，一个kubefile中可以有多个CMD指令。

> 命令格式：CMD {command args ...}

使用样例：

例如上面示例中,使用 kubectl 命令安装一个kubernetes的dashboard。

`CMD kubectl apply -f recommended.yaml`

## sealer build 实际操作

Kubefile

```shell
FROM kubernetes:v1.19.8
ARG Version=4.0.0 
RUN echo ${Version}
COPY mysql.tar images
COPY imageList manifests
COPY traefik charts
COPY shell.yaml plugins
COPY recommended.yaml manifests
CMD kubectl apply -f manifests/recommended.yaml
CMD helm install mytest charts/traefik
```

build cmd line:

sealer build -t my-dashboard:v1 .

run cmd line:

sealer run -m 172.16.0.227 -p Seadent123 my-dashboard:v1