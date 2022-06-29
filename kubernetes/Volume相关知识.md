# 卷的分类
## Volume

## PersistentVolume(PV)
PV 为 Kubernete 集群提供了一个如何提供并且使用存储的抽象，与它一起被引入的另一个对象就是 PersistentVolumeClaim(PVC)，这两个对象之间的关系与节点和 Pod 之间的关系差不多：
![](./PC%E4%B8%8EPVC.png)
PersistentVolume 是集群中的一种被管理员分配的存储资源<br>
PersistentVolumeClaim 表示用户对存储资源的申请，它与 Pod 非常相似，PVC 消耗了持久卷资源，而 Pod 消耗了节点上的 CPU 和内存等物理资源<br>

# Attach 与 Mount
创建一个远程块存储，相当于创建了一个磁盘，称为Attach<br>
由Volume Controller负责维护，不断地检查 每个Pod对应的PV和所在的宿主机的挂载情况。可以理解为创建了一块NFS磁盘<br>
为了使用这块磁盘，还需要将这个磁盘设备挂载到宿主机的挂载点，称为Mount<br>

如果是已经有NFS磁盘，可以省略Attach.<br>
同样，删除PV的时候，也需要Umount和Dettach两个阶段处理<br>



##  总结
1、PVC和PV相当于面向对象的接口和实现

2、用户创建的Pod声明了PVC，K8S会找一个PV配对，如果没有PV，就去找对应的StorageClass，帮它创建一个PV，然后和PVC完成绑定

3、新创建的PV，要经过Master节点Attach为宿主机创建远程磁盘，再经过每个节点kubelet组件把Attach的远程磁盘Mount到宿主机目录