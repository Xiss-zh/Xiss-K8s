# rook-ceph部署

### 一、rook先决条件

```shell
# 1、Kubernetes v1.21或更高版本
# 2、无分区或格式化的文件系统的原始磁盘
# 3、或者无格式化文件系统的LVM卷
# 4、内核需加载RBD模块（modinfo rbd，一般默认已经加载）
# 5、为了后期支持Cephfs特性，建议将内核升级到5.4以上
# 6、最少三个node节点
```

![image-20230509103740815](D:\Github\Xiss-K8s\rook\png\image-20230509103740815.png)

![image-20230509103954862](D:\Github\Xiss-K8s\rook\png\image-20230509103954862.png)

### 二、rook准备

#### 1、克隆rook项目到本地

```shell
# 克隆rook项目到本地
git clone --single-branch --branch v1.11.5 https://github.com/rook/rook.git

```

![image-20230509104339401](D:\Github\Xiss-K8s\rook\png\image-20230509104339401.png)

```shell
# 拷贝基本集群配置文件到其它文件夹
cd rook/deploy/examples

# rook自定义的公共资源清单
crds.yaml
# rook公共配置、服务账号等
common.yaml
# k8s操作员清单，监视rook资源变更情况去创建、更新或删除
operator.yaml
# rook集群清单，定义集群的各种属性
cluster.yaml
# 涉及到的镜像版本，部分镜像国内下载不了，需通过阿里云下载
images.txt
# ceph的管理工具
toolbox.yaml
# 官方dashboard，一般只是查看，不建议操作
dashboard-external-https.yaml
```

#### 2、下载相关镜像

```shell
# docker：下载镜像并重新打标
tee ./rook-images.sh <<'EOF'
#!/bin/bash
images=(
	csi-attacher:v4.1.0
	csi-node-driver-registrar:v2.7.0
	csi-provisioner:v3.4.0
	csi-resizer:v1.7.0
	csi-snapshotter:v6.2.1

)

for imageName in ${images[@]} ; do
	docker pull registry.aliyuncs.com/google_containers/$imageName
	docker tag registry.aliyuncs.com/google_containers/$imageName registry.k8s.io/sig-storage/$imageName
	docker rmi registry.aliyuncs.com/google_containers/$imageName
done

docker pull quay.io/ceph/ceph:v16.2.11
docker pull quay.io/cephcsi/cephcsi:v3.8.0
docker pull quay.io/csiaddons/k8s-sidecar:v0.5.0
docker pull rook/ceph:v1.11.5

EOF

# 执行下载镜像
chmod +x ./rook-images.sh && ./rook-images.sh
```



![image-20230509140513727](D:\Github\Xiss-K8s\rook\png\image-20230509140513727.png)

### 三、rook部署

```shell
# 创建rook相关清单资源
kubectl create -f common.yaml -f crds.yaml -f operator.yaml

# cluster.yaml为定义集群创建，简单点可以默认，需要自定义请查看官方手册
# 默认集群会创建mon三个pod，mgr两个pod，osd自动分配node节点未使用的磁盘
kubectl create -f cluster.yaml

# 创建一个ceph管理工具
kubectl create -f toolbox.yaml

# 至此rook完成创建ceph集群，可通过访问ceph-tools查看ceph集群状态
kubectl exec -it -n rook-ceph rook-ceph-tools-54bdbfc7b7-pm95d -- bash -c "ceph -s"

# 创建访问官方dashboard的服务
kubectl create -f dashboard-external-https.yaml

# 查询对外访问的接口地址
kubectl get svc -n rook-ceph
# 例子，NodePort调整有点问题，需配置nginx
https://192.168.10.231:32093/

# 查询dashboard的密码，默认账号admin
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo


#
```

![image-20230509152850615](D:\Github\Xiss-K8s\rook\png\image-20230509152850615.png)

![image-20230509153340111](D:\Github\Xiss-K8s\rook\png\image-20230509153340111.png)

![image-20230509154447867](D:\Github\Xiss-K8s\rook\png\image-20230509154447867.png)

### 四、创建storageclass

```shell
# 进入rook/deploy/examples/csi/rbd目录
kubectl create -f storageclass.yaml

# 查看storageclass和ceph存储池
kubectl get sc
kubectl exec -it -n rook-ceph rook-ceph-tools-54bdbfc7b7-pm95d -- bash -c "ceph osd pool ls"
```

![image-20230509161211445](D:\Github\Xiss-K8s\rook\png\image-20230509161211445.png)



```

```

![image-20230512165805183](C:\Users\zhangxing\AppData\Roaming\Typora\typora-user-images\image-20230512165805183.png)

![image-20230515105530359](C:\Users\zhangxing\AppData\Roaming\Typora\typora-user-images\image-20230515105530359.png)
