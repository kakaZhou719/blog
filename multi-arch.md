# 多架构镜像

## build
对于多架构镜像：

1. 只要增加platform flag 就会创建manifest

sealer build  -f Kubefile -t multi-images:v1 --platform linux/amd64,linux/arm64
sealer build  -f Kubefile -t multi-images:v1 --platform linux/amd64
sealer build  -f Kubefile -t multi-images:v1 --platform linux/arm64


## push
对于多架构镜像： 默认 push所有镜像以及manifest

sealer push multi-images:v1

## pull
对于多架构镜像：下载对应架构的镜像，默认pull系统架构对应的image

sealer pull multi-images:v1 --platform linux/arm64


## save
针对单个镜像，默认 save 系统架构对应的image
sealer save multi-images:v1

## load
不涉及

## inspect 
针对单个镜像，默认 inspect 系统架构对应的image
sealer inspect multi-images:v1

## rmi
针对单个镜像，只删除 manifest，不会删除对应的image
sealer rmi multi-images:v1


