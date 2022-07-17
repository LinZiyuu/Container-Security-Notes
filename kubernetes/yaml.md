

# CustomResourceDefinition
## 重要字段
### (1)apiVersion
### (2)kind:

​	可分为三类:

​		(1)Object:Pod ReplicationController Service Namespace Node(通常表示系统中的持久实体)

​		(2)Lists:PodList ServiceList NodeList(通常表示一种资源的集合)

​		(3)Simple:binding(将一些用户请求的资源比如Pod，PVC绑定到Node，PV上) status scale (proxy portforward)(用于一些特定操作或者绑定一些非持久的实体)

### (3)metadata：每一个Object kind都得有一个metadata

​	namespace：默认为default

​	name：是当前namespace里唯一的

​	uid：在时空上唯一，用于区分已经被删除或者重建的对象

​	resourceVersion：用于表示对象内部的版本

​	generation：

​	creationTimestamp：对象创建的日期和时间

​	deletionTimestamp：表示资源被删除的时间(由server创建)

​	labels：用于组织和分类对象(比如用户用labels查询pod)

​	annotation：外部第三方工具使用它来检索或修改此对象的metadata

### (4)spec：对象的期望状态(包括container storage volumes信息)
### (5)status：对象的当前状态



# yaml中的主要参数

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```



# Volume 类型

- **emptyDir**：当 Pod 分派到某个 Node 上时，`emptyDir` 卷会被创建，并且在 Pod 在该节点上运行期间，卷一直存在。 就像其名称表示的那样，卷最初是空的。 尽管 Pod 中的容器挂载 `emptyDir` 卷的路径可能相同也可能不同，这些容器都可以读写 `emptyDir` 卷中相同的文件。 当 Pod 因为某些原因被从节点上删除时，`emptyDir` 卷中的数据也会被永久删除。(容器崩溃并**不**会导致 Pod 被从节点上移除，因此容器崩溃期间 `emptyDir` 卷中的数据是安全的。)

  **emptyDir.medium** (string)

  ​	medium 表示此目录应使用哪种类别的存储介质。默认为 ""，这意味着使用节点的默认介质。 必须是空字符串（默认值）或 Memory。

  **emptyDir.sizeLimit** (Quantity)

  ​	sizeLimit 是这个 EmptyDir 卷所需的本地存储总量。这个大小限制也适用于内存介质。 EmptyDir 的内存介质最大使用量将是此处指定的 sizeLimit 与 Pod 中所有容器内存限制总和这两个值之间的最小值。 默认为 nil，这意味着限制未被定义。

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-pd
  spec:
    containers:
    - image: gcr.io/google_containers/test-webserver
      name: test-container
      volumeMounts:
      - mountPath: /cache
        name: cache-volume
    volumes:
    - name: cache-volume
    #volumeType
      emptyDir: {}
  ```

  

- **hostPath**:`hostPath` 卷能将主机节点文件系统上的文件或目录挂载到你的 Pod 中。 这种卷通常用于系统代理或允许查看主机的其他特权操作。

  **hostPath.path** (string)，必需

  ​	目录在主机上的路径。如果该路径是一个符号链接，则它将沿着链接指向真实路径。

  **hostPath.type** (string)

  HostPath 卷的类型。默认为 ""。

  type值类型作用：

  | 取值                | 行为                                                         |
  | :------------------ | :----------------------------------------------------------- |
  |                     | 空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。 |
  | `DirectoryOrCreate` | 如果在给定路径上什么都不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息。 |
  | `Directory`         | 在给定路径上必须存在的目录。                                 |
  | `FileOrCreate`      | 如果在给定路径上什么都不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权。 |
  | `File`              | 在给定路径上必须存在的文件。                                 |
  | `Socket`            | 在给定路径上必须存在的 UNIX 套接字。                         |
  | `CharDevice`        | 在给定路径上必须存在的字符设备。                             |
  | `BlockDevice`       | 在给定路径上必须存在的块设备。                               |

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-pd
  spec:
    containers:
    - image: k8s.gcr.io/test-webserver
      name: test-container
      volumeMounts:
      - mountPath: /test-pd
        name: test-volume
    volumes:
    - name: test-volume
      hostPath:
        # 宿主上目录位置
        path: /data
        # 此字段为可选
        type: Directory
  ```

  

- gcePersistentDisk:gcePersistentDisk 可以挂载 GCE 上的永久磁盘到容器，需要 Kubernetes 运行在 GCE 的 VM 中。

  ```yaml
  volumes:
    - name: test-volume
      # This GCE PD must already exist.
      gcePersistentDisk:
        pdName: my-data-disk
        fsType: ext4
  ```

  

- nfs：NFS 是 Network File System 的缩写，即网络文件系统。Kubernetes 中通过简单地配置就可以挂载 NFS 到 Pod 中，而 NFS 中的数据是可以永久保存的，同时 NFS 支持同时写操作。

  ```yaml
  volumes:
  - name: nfs
    nfs:
      # FIXME: use the right hostname
      server: 10.254.234.223
      path: "/"
  ```

  

- iscsi

- glusterfs

- rbd

- cephfs

- secret

- persistentVolumeClaim

- downwardAPI

- azureFileVolume

- PortworxVolume

- ScaleIO

- FlexVolume

- local

- 使用subPath



参考：

https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#types-kinds