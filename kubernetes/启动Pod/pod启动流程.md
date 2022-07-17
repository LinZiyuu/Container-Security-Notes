# 
创建一个pod kubectl apply -f xxx.yaml

(1)客户端发送的HTTP请求已经通过校验（API Server的身份认证、鉴权和准入控制）
(2)已经把资源对象信息持久化到 etcd 中
(3)master节点的调度器已经把pod调度到合适的节点上
(4)接下的步骤就涉及到在工作节点上真正创建 pod,这部分由kubelet组件负责。

# 源码分析

![](pod%E5%90%AF%E5%8A%A8.png)