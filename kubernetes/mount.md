
## Volume manager
```go
//in kubernetes/pkg/kubelet/volumemanager/volume_manager.go
//常量的定义
const (
    // reconcilerLoopSleepPeriod is the amount of time the reconciler loop waits
    // between successive executions
    reconcilerLoopSleepPeriod = 100 * time.Millisecond

    // desiredStateOfWorldPopulatorLoopSleepPeriod is the amount of time the
    // DesiredStateOfWorldPopulator loop waits between successive executions
    desiredStateOfWorldPopulatorLoopSleepPeriod = 100 * time.Millisecond

    // desiredStateOfWorldPopulatorGetPodStatusRetryDuration is the amount of
    // time the DesiredStateOfWorldPopulator loop waits between successive pod
    // cleanup calls (to prevent calling containerruntime.GetPodStatus too
    // frequently).
    desiredStateOfWorldPopulatorGetPodStatusRetryDuration = 2 * time.Second

    // podAttachAndMountTimeout is the maximum amount of time the
    // WaitForAttachAndMount call will wait for all volumes in the specified pod
    // to be attached and mounted. Even though cloud operations can take several
    // minutes to complete, we set the timeout to 2 minutes because kubelet
    // will retry in the next sync iteration. This frees the associated
    // goroutine of the pod to process newer updates if needed (e.g., a delete
    // request to the pod).
    // Value is slightly offset from 2 minutes to make timeouts due to this
    // constant recognizable.
    podAttachAndMountTimeout = 2*time.Minute + 3*time.Second

    // podAttachAndMountRetryInterval is the amount of time the GetVolumesForPod
    // call waits before retrying
    podAttachAndMountRetryInterval = 300 * time.Millisecond

    // waitForAttachTimeout is the maximum amount of time a
    // operationexecutor.Mount call will wait for a volume to be attached.
    // Set to 10 minutes because we've seen attach operations take several
    // minutes to complete for some volume plugins in some cases. While this
    // operation is waiting it only blocks other operations on the same device,
    // other devices are not affected.
    waitForAttachTimeout = 10 * time.Minute
)

type VolumeManager interface {
    //启动volume manager
	
    // Starts the volume manager and all the asynchronous loops that it controls
	Run(sourcesReady config.SourcesReady, stopCh <-chan struct{})

    //在启动pod之前会先进入阻塞等待需要被Attach或者Mount的Volume被处理
	
    // WaitForAttachAndMount processes the volumes referenced in the specified
	// pod and blocks until they are all attached and mounted (reflected in
	// actual state of the world).
	// An error is returned if all volumes are not attached and mounted within
	// the duration defined in podAttachAndMountTimeout.
	WaitForAttachAndMount(pod *v1.Pod) error

    //在启动pod之前会先进入阻塞等待需要被Unmount的Volume被处理
	
    // WaitForUnmount processes the volumes referenced in the specified
	// pod and blocks until they are all unmounted (reflected in the actual
	// state of the world).
	// An error is returned if all volumes are not unmounted within
	// the duration defined in podAttachAndMountTimeout.
	WaitForUnmount(pod *v1.Pod) error

    //获得已经mounted在pod上的volume信息以map的形式返回
    //key 是OuterVolumeSpecName 
    //(i.e.pod.Spec.Volumes[x].Name)

	// GetMountedVolumesForPod returns a VolumeMap containing the volumes
	// referenced by the specified pod that are successfully attached and
	// mounted. The key in the map is the OuterVolumeSpecName (i.e.
	// pod.Spec.Volumes[x].Name). It returns an empty VolumeMap if pod has no
	// volumes.
	GetMountedVolumesForPod(podName types.UniquePodName) container.VolumeMap

    //获得可能/已经mount或者attatch在Pod上的volume信息,以map形式返回

	// GetPossiblyMountedVolumesForPod returns a VolumeMap containing the volumes
	// referenced by the specified pod that are either successfully attached
	// and mounted or are "uncertain", i.e. a volume plugin may be mounting
	// them right now. The key in the map is the OuterVolumeSpecName (i.e.
	// pod.Spec.Volumes[x].Name). It returns an empty VolumeMap if pod has no
	// volumes.
	GetPossiblyMountedVolumesForPod(podName types.UniquePodName) container.VolumeMap

    //获得Pod额外需要依赖的volume信息以数组形式返回？

	// GetExtraSupplementalGroupsForPod returns a list of the extra
	// supplemental groups for the Pod. These extra supplemental groups come
	// from annotations on persistent volumes that the pod depends on.
	GetExtraSupplementalGroupsForPod(pod *v1.Pod) []int64

    //获取正在使用中的volume信息以list形式返回

	// GetVolumesInUse returns a list of all volumes that implement the volume.Attacher
	// interface and are currently in use according to the actual and desired
	// state of the world caches. A volume is considered "in use" as soon as it
	// is added to the desired state of world, indicating it *should* be
	// attached to this node and remains "in use" until it is removed from both
	// the desired state of the world and the actual state of the world, or it
	// has been unmounted (as indicated in actual state of world).
	GetVolumesInUse() []v1.UniqueVolumeName

    //当actual state of world被完全同步的情况返回true

	// ReconcilerStatesHasBeenSynced returns true only after the actual states in reconciler
	// has been synced at least once after kubelet starts so that it is safe to update mounted
	// volume list retrieved from actual state.
	ReconcilerStatesHasBeenSynced() bool

    //判断这个volume是否被attach

	// VolumeIsAttached returns true if the given volume is attached to this
	// node.
	VolumeIsAttached(volumeName v1.UniqueVolumeName) bool

    //标记一个volume的状态为 in use 有说明用？？
	// Marks the specified volume as having successfully been reported as "in
	// use" in the nodes's volume status.
	MarkVolumesAsReportedInUse(volumesReportedAsInUse []v1.UniqueVolumeName)
}



// volumeManager implements the VolumeManager interface
type volumeManager struct {
    
    //DesiredStateOfWorldPopulator是什么？

    // kubeClient is the kube API client used by DesiredStateOfWorldPopulator to
    // communicate with the API server to fetch PV and PVC objects
    
    //这是什么用法？？
    kubeClient clientset.Interface

    //

    // volumePluginMgr is the volume plugin manager used to access volume
    // plugins. It must be pre-initialized.
    //volume库
    volumePluginMgr *volume.VolumePluginMgr

    //desiredStateOfWorld记录哪些volume需要被attach,哪些pod需要引用了卷 期望达到的情况

    // desiredStateOfWorld is a data structure containing the desired state of
    // the world according to the volume manager: i.e. what volumes should be
    // attached and which pods are referencing the volumes).
    // The data structure is populated by the desired state of the world
    // populator using the kubelet pod manager.
    //cache库
    desiredStateOfWorld cache.DesiredStateOfWorld

    //actualStateOfWorld记录记录哪些volume被attach,哪些pod引用了卷 真实情况

    // actualStateOfWorld is a data structure containing the actual state of
    // the world according to the manager: i.e. which volumes are attached to
    // this node and what pods the volumes are mounted to.
    // The data structure is populated upon successful completion of attach,
    // detach, mount, and unmount actions triggered by the reconciler.
    actualStateOfWorld cache.ActualStateOfWorld

    //操作执行器,具体怎么实现还没看，是operation库

    // operationExecutor is used to start asynchronous attach, detach, mount,
    // and unmount operations.
    operationExecutor operationexecutor.OperationExecutor

    //reconciler的作用是按desiredStateOfWorld来同步volume配置操作

    // reconciler runs an asynchronous periodic loop to reconcile the
    // desiredStateOfWorld with the actualStateOfWorld by triggering attach,
    // detach, mount, and unmount operations using the operationExecutor.
    reconciler reconciler.Reconciler

    //dswp使用PodManager按照desiredStateOfWorld来配置

    // desiredStateOfWorldPopulator runs an asynchronous periodic loop to
    // populate the desiredStateOfWorld using the kubelet PodManager.
    desiredStateOfWorldPopulator populator.DesiredStateOfWorldPopulator

    //底下两个不知道啥作用

    // csiMigratedPluginManager keeps track of CSI migration status of plugins
    csiMigratedPluginManager csimigration.PluginManager

    // intreeToCSITranslator translates in-tree volume specs to CSI
    intreeToCSITranslator csimigration.InTreeToCSITranslator
}
```

## Volume manager 接口说明
(1)运行在kubelet 里让存储Ready的部件，主要是mount/unmount（attach/detach可选）
(2)pod调度到这个node上后才会有卷的相应操作，所以它的触发端是kubelet（严格讲是kubelet里的pod manager），根据Pod Manager里pod spec里申明的存储来触发卷的挂载操作
(3)Kubelet会监听到调度到该节点上的pod声明，会把pod缓存到Pod Manager中，VolumeManager通过Pod Manager获取PV/PVC的状态，并进行分析出具体的attach/detach、mount/umount, 操作然后调用plugin进行相应的业务处理

## VolumeManger 结构体
volumeManager结构体实现了VolumeManager接口，主要有两个需要注意：
(1)desiredStateOfWorld：预期状态，volume需要被attach，哪些pods引用这个volume
(2)actualStateOfWorld：实际状态，volume已经被atttach到哪个node，哪个pod mount volume

## desiredStateOfWorld 和 actualStateOfWorld
desiredStateOfWorld为理想的volume情况，它主要是根据podManger获取所有的Pod信息，从中提取Volume信息。

actualStateOfWorld则是实际的volume情况。

desiredStateOfWorldPopulator通过podManager去构建desiredStateOfWorld。

reconciler的工作主要是比较actualStateOfWorld和desiredStateOfWorld的差别，然后进行volume的创建、删除和修改，最后使二者达到一致。

## 流程
(1)新建
NewVolumeManager中主要构造了几个volume控制<br>
volumePluginMgr 和 csiMigratedPluginManager <br>
desiredStateOfWorldPopulator <br>
reconciler<br>
```go
//in kubernetes\pkg\kubelet\kubelet.go


// NewMainKubelet instantiates a new Kubelet object along with all the required internal modules.
// No initialization of Kubelet and its modules should happen here.
func NewMainKubelet(){

    // ......

    // setup volumeManager
    klet.volumeManager = volumemanager.NewVolumeManager(
        //各种初始化信息没细看
        kubeCfg.EnableControllerAttachDetach,
        nodeName,
        klet.podManager,
        klet.statusManager,
        klet.kubeClient,
        klet.volumePluginMgr,
        klet.containerRuntime,
        kubeDeps.Mounter,
        kubeDeps.HostUtil,
        klet.getPodsDir(),
        kubeDeps.Recorder,
        experimentalCheckNodeCapabilitiesBeforeMount,
        keepTerminatedPodVolumes,
        volumepathhandler.NewBlockVolumePathHandler())

    // ......
}


//in kubernetes/pkg/kubelet/volumanager/volume_manager.go

// NewVolumeManager returns a new concrete instance implementing the
// VolumeManager interface.

// kubeClient - kubeClient is the kube API client used by DesiredStateOfWorldPopulator
//   to communicate with the API server to fetch PV and PVC objects
// volumePluginMgr - the volume plugin manager used to access volume plugins.
//   Must be pre-initialized.
func NewVolumeManager(
	controllerAttachDetachEnabled bool,
	nodeName k8stypes.NodeName,
	podManager pod.Manager,
	podStateProvider podStateProvider,
	kubeClient clientset.Interface,
	volumePluginMgr *volume.VolumePluginMgr,
	kubeContainerRuntime container.Runtime,
	mounter mount.Interface,
	hostutil hostutil.HostUtils,
	kubeletPodsDir string,
	recorder record.EventRecorder,
	keepTerminatedPodVolumes bool,
	blockVolumePathHandler volumepathhandler.BlockVolumePathHandler) VolumeManager {

    vm := &volumeManager{
        kubeClient:          kubeClient,
        volumePluginMgr:     volumePluginMgr,
        desiredStateOfWorld: cache.NewDesiredStateOfWorld(volumePluginMgr),
        actualStateOfWorld:  cache.NewActualStateOfWorld(nodeName, volumePluginMgr),
        operationExecutor: operationexecutor.NewOperationExecutor(operationexecutor.NewOperationGenerator(
            kubeClient,
            volumePluginMgr,
            recorder,
            checkNodeCapabilitiesBeforeMount,
            blockVolumePathHandler)),
    }

    intreeToCSITranslator := csitrans.New()
    csiMigratedPluginManager := csimigration.NewPluginManager(intreeToCSITranslator)

    vm.intreeToCSITranslator = intreeToCSITranslator
    vm.csiMigratedPluginManager = csiMigratedPluginManager
    vm.desiredStateOfWorldPopulator = populator.NewDesiredStateOfWorldPopulator(
        kubeClient,
        desiredStateOfWorldPopulatorLoopSleepPeriod,
        desiredStateOfWorldPopulatorGetPodStatusRetryDuration,
        podManager,
        podStatusProvider,
        vm.desiredStateOfWorld,
        vm.actualStateOfWorld,
        kubeContainerRuntime,
        keepTerminatedPodVolumes,
        csiMigratedPluginManager,
        intreeToCSITranslator,
        volumePluginMgr)
    vm.reconciler = reconciler.NewReconciler(
        kubeClient,
        controllerAttachDetachEnabled,
        reconcilerLoopSleepPeriod,
        waitForAttachTimeout,
        nodeName,
        vm.desiredStateOfWorld,
        vm.actualStateOfWorld,
        vm.desiredStateOfWorldPopulator.HasAddedPods,
        vm.operationExecutor,
        mounter,
        hostutil,
        volumePluginMgr,
        kubeletPodsDir)

    return vm
}

```

desiredStateOfWorld和actualStateOfWorld源码:

```go

//in kubenetes/pkg/controller/volume/attachdetach/cache/desired_state_of_world.go

//实例化
// NewDesiredStateOfWorld returns a new instance of DesiredStateOfWorld.
func NewDesiredStateOfWorld(volumePluginMgr *volume.VolumePluginMgr) DesiredStateOfWorld {
	return &desiredStateOfWorld{
		nodesManaged:    make(map[k8stypes.NodeName]nodeManaged),
		volumePluginMgr: volumePluginMgr,
	}
}

type desiredStateOfWorld struct {
	// nodesManaged is a map containing the set of nodes managed by the attach/
	// detach controller. The key in this map is the name of the node and the
	// value is a node object containing more information about the node.


    //key是nodeName value是相关信息

	nodesManaged map[k8stypes.NodeName]nodeManaged
	// volumePluginMgr is the volume plugin manager used to create volume
	// plugin objects.

	volumePluginMgr *volume.VolumePluginMgr
	
    //这个是什么作用？？
    sync.RWMutex
}

//in kubenetes/pkg/controller/volume/attachdetach/cache/actual_state_of_world.go

// NewActualStateOfWorld returns a new instance of ActualStateOfWorld.
func NewActualStateOfWorld(volumePluginMgr *volume.VolumePluginMgr) ActualStateOfWorld {
	return &actualStateOfWorld{
		attachedVolumes:        make(map[v1.UniqueVolumeName]attachedVolume),
		nodesToUpdateStatusFor: make(map[types.NodeName]nodeToUpdateStatusFor),
		volumePluginMgr:        volumePluginMgr,
	}
}

type actualStateOfWorld struct {
	// attachedVolumes is a map containing the set of volumes the attach/detach
	// controller believes to be successfully attached to the nodes it is
	// managing. The key in this map is the name of the volume and the value is
	// an object containing more information about the attached volume.


    //存储的是已经成功attach到node上的volume
	attachedVolumes map[v1.UniqueVolumeName]attachedVolume



	// nodesToUpdateStatusFor is a map containing the set of nodes for which to
	// update the VolumesAttached Status field. The key in this map is the name
	// of the node and the value is an object containing more information about
	// the node (including the list of volumes to report attached).

    //存储更新status失败的node信息
	nodesToUpdateStatusFor map[types.NodeName]nodeToUpdateStatusFor

	// volumePluginMgr is the volume plugin manager used to create volume
	// plugin objects.
	volumePluginMgr *volume.VolumePluginMgr

	sync.RWMutex
}
```

NewOperationExecutor源码:
```go

// in kubernetes/pkg/kubelet/pluginmanager/operationexecutor/operation_executor.go

// NewOperationExecutor returns a new instance of OperationExecutor.
func NewOperationExecutor(
	operationGenerator OperationGenerator) OperationExecutor {

	return &operationExecutor{
		pendingOperations:  goroutinemap.NewGoRoutineMap(true /* exponentialBackOffOnError */),
		operationGenerator: operationGenerator,
	}
}

```
NewpluginManager源码:
```go
// in kubernetes/pkg/kubelet/pluginmanager/plugin_manager.go
func NewPluginManager(
	sockDir string,
	recorder record.EventRecorder) PluginManager {
	asw := cache.NewActualStateOfWorld()
	dsw := cache.NewDesiredStateOfWorld()
	reconciler := reconciler.NewReconciler(
		operationexecutor.NewOperationExecutor(
			operationexecutor.NewOperationGenerator(
				recorder,
			),
		),
		loopSleepDuration,
		dsw,
		asw,
	)

	pm := &pluginManager{
		desiredStateOfWorldPopulator: pluginwatcher.NewWatcher(
			sockDir,
			dsw,
		),
		reconciler:          reconciler,
		desiredStateOfWorld: dsw,
		actualStateOfWorld:  asw,
	}
	return pm
}
```


NewDesiredStateOfWorldPopulator源码:
```go

//in kubernetes/pkg/controller/volume/attachdetach/populator/desired_state_of_world_populator.go

// NewDesiredStateOfWorldPopulator returns a new instance of
// DesiredStateOfWorldPopulator.
//
// kubeClient - used to fetch PV and PVC objects from the API server
// loopSleepDuration - the amount of time the populator loop sleeps between
//     successive executions
// podManager - the kubelet podManager that is the source of truth for the pods
//     that exist on this host
// desiredStateOfWorld - the cache to populate
func NewDesiredStateOfWorldPopulator(
    kubeClient clientset.Interface,
    loopSleepDuration time.Duration,
    getPodStatusRetryDuration time.Duration,
    podManager pod.Manager,
    podStatusProvider status.PodStatusProvider,
    desiredStateOfWorld cache.DesiredStateOfWorld,
    actualStateOfWorld cache.ActualStateOfWorld,
    kubeContainerRuntime kubecontainer.Runtime,
    keepTerminatedPodVolumes bool,
    csiMigratedPluginManager csimigration.PluginManager,
    intreeToCSITranslator csimigration.InTreeToCSITranslator,
    volumePluginMgr *volume.VolumePluginMgr) DesiredStateOfWorldPopulator {
    return &desiredStateOfWorldPopulator{
        kubeClient:                kubeClient,
        loopSleepDuration:         loopSleepDuration,
        getPodStatusRetryDuration: getPodStatusRetryDuration,
        podManager:                podManager,
        podStatusProvider:         podStatusProvider,
        desiredStateOfWorld:       desiredStateOfWorld,
        actualStateOfWorld:        actualStateOfWorld,
        pods: processedPods{
            processedPods: make(map[volumetypes.UniquePodName]bool)},
        kubeContainerRuntime:     kubeContainerRuntime,
        keepTerminatedPodVolumes: keepTerminatedPodVolumes,
        hasAddedPods:             false,
        hasAddedPodsLock:         sync.RWMutex{},
        csiMigratedPluginManager: csiMigratedPluginManager,
        intreeToCSITranslator:    intreeToCSITranslator,
        volumePluginMgr:          volumePluginMgr,
    }
}

type desiredStateOfWorldPopulator struct {
    kubeClient                clientset.Interface
    loopSleepDuration         time.Duration
    getPodStatusRetryDuration time.Duration
    podManager                pod.Manager
    podStatusProvider         status.PodStatusProvider
    desiredStateOfWorld       cache.DesiredStateOfWorld
    actualStateOfWorld        cache.ActualStateOfWorld
    pods                      processedPods
    kubeContainerRuntime      kubecontainer.Runtime
    timeOfLastGetPodStatus    time.Time
    keepTerminatedPodVolumes  bool
    hasAddedPods              bool
    hasAddedPodsLock          sync.RWMutex
    csiMigratedPluginManager  csimigration.PluginManager
    intreeToCSITranslator     csimigration.InTreeToCSITranslator
    volumePluginMgr           *volume.VolumePluginMgr
}



// NewReconciler returns a new instance of Reconciler.
//
// controllerAttachDetachEnabled - if true, indicates that the attach/detach
//   controller is responsible for managing the attach/detach operations for
//   this node, and therefore the volume manager should not
// loopSleepDuration - the amount of time the reconciler loop sleeps between
//   successive executions
// waitForAttachTimeout - the amount of time the Mount function will wait for
//   the volume to be attached
// nodeName - the Name for this node, used by Attach and Detach methods
// desiredStateOfWorld - cache containing the desired state of the world
// actualStateOfWorld - cache containing the actual state of the world
// populatorHasAddedPods - checker for whether the populator has finished
//   adding pods to the desiredStateOfWorld cache at least once after sources
//   are all ready (before sources are ready, pods are probably missing)
// operationExecutor - used to trigger attach/detach/mount/unmount operations
//   safely (prevents more than one operation from being triggered on the same
//   volume)
// mounter - mounter passed in from kubelet, passed down unmount path
// hostutil - hostutil passed in from kubelet
// volumePluginMgr - volume plugin manager passed from kubelet
func NewReconciler(
    kubeClient clientset.Interface,
    controllerAttachDetachEnabled bool,
    loopSleepDuration time.Duration,
    waitForAttachTimeout time.Duration,
    nodeName types.NodeName,
    desiredStateOfWorld cache.DesiredStateOfWorld,
    actualStateOfWorld cache.ActualStateOfWorld,
    populatorHasAddedPods func() bool,
    operationExecutor operationexecutor.OperationExecutor,
    mounter mount.Interface,
    hostutil hostutil.HostUtils,
    volumePluginMgr *volumepkg.VolumePluginMgr,
    kubeletPodsDir string) Reconciler {
    return &reconciler{
        kubeClient:                    kubeClient,
        controllerAttachDetachEnabled: controllerAttachDetachEnabled,
        loopSleepDuration:             loopSleepDuration,
        waitForAttachTimeout:          waitForAttachTimeout,
        nodeName:                      nodeName,
        desiredStateOfWorld:           desiredStateOfWorld,
        actualStateOfWorld:            actualStateOfWorld,
        populatorHasAddedPods:         populatorHasAddedPods,
        operationExecutor:             operationExecutor,
        mounter:                       mounter,
        hostutil:                      hostutil,
        volumePluginMgr:               volumePluginMgr,
        kubeletPodsDir:                kubeletPodsDir,
        timeOfLastSync:                time.Time{},
    }
}

type reconciler struct {
    kubeClient                    clientset.Interface
    controllerAttachDetachEnabled bool
    loopSleepDuration             time.Duration
    waitForAttachTimeout          time.Duration
    nodeName                      types.NodeName
    desiredStateOfWorld           cache.DesiredStateOfWorld
    actualStateOfWorld            cache.ActualStateOfWorld
    populatorHasAddedPods         func() bool
    operationExecutor             operationexecutor.OperationExecutor
    mounter                       mount.Interface
    hostutil                      hostutil.HostUtils
    volumePluginMgr               *volumepkg.VolumePluginMgr
    kubeletPodsDir                string
    timeOfLastSync                time.Time
}
```

(2)启动
kl.volumeManager.Run<br>
启动子模块有:<br>
(2.1)如果有volumePlugin（默认安装时没有插件），启动volumePluginMgr<br>
(2.2)启动 desiredStateOfWorldPopulator：从apiserver同步到的pod信息，更新DesiredStateOfWorld<br>
    findAndAddNewPods()<br>
    findAndRemoveDeletedPods() 每隔dswp.getPodStatusRetryDuration时长，进行findAndRemoveDeletedPods()<br>
(2.3)启动 reconciler：预期状态和实际状态的协调者，负责调整实际状态至预期状态<br>

kubelet、volumeManager  VolumePluginMgr desiredStateOfWorldPopulator Run 源码:

```go

// in/kubernetes/pkg/kubelet/kubelet.go

// Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	if kl.logServer == nil {
		kl.logServer = http.StripPrefix("/logs/", http.FileServer(http.Dir("/var/log/")))
	}
	if kl.kubeClient == nil {
		klog.InfoS("No API server defined - no node status update will be sent")
	}

	// Start the cloud provider sync manager
	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		klog.ErrorS(err, "Failed to initialize internal modules")
		os.Exit(1)
	}

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		// Introduce some small jittering to ensure that over time the requests won't start
		// accumulating at approximately the same time from the set of nodes due to priority and
		// fairness effect.
		go wait.JitterUntil(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
		go kl.fastStatusUpdateOnce()

		// start syncing lease
		go kl.nodeLeaseController.Run(wait.NeverStop)
	}
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Set up iptables util rules
	if kl.makeIPTablesUtilChains {
		kl.initNetworkUtil()
	}

	// Start component sync loops.
	kl.statusManager.Start()

	// Start syncing RuntimeClasses if enabled.
	if kl.runtimeClassManager != nil {
		kl.runtimeClassManager.Start(wait.NeverStop)
	}

	// Start the pod lifecycle event generator.
	kl.pleg.Start()
	kl.syncLoop(updates, kl)
}

//in /kubernetes/pkg/kubelet/volumemanager/volume_manager.go

func (vm *volumeManager) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
    defer runtime.HandleCrash()

    if vm.kubeClient != nil {
        // start informer for CSIDriver
        go vm.volumePluginMgr.Run(stopCh)
    }

    go vm.desiredStateOfWorldPopulator.Run(sourcesReady, stopCh)
    klog.V(2).Infof("The desired_state_of_world populator starts")

    klog.Infof("Starting Kubelet Volume Manager")
    go vm.reconciler.Run(stopCh)

    metrics.Register(vm.actualStateOfWorld, vm.desiredStateOfWorld, vm.volumePluginMgr)

    <-stopCh
    klog.Infof("Shutting down Kubelet Volume Manager")
}

//in /kubernetes/pkg/volume/plugins.go
func (pm *VolumePluginMgr) Run(stopCh <-chan struct{}) {
	kletHost, ok := pm.Host.(KubeletVolumeHost)
	if ok {
		// start informer for CSIDriver
		informerFactory := kletHost.GetInformerFactory()
		informerFactory.Start(stopCh)
		informerFactory.WaitForCacheSync(stopCh)
	}
}

//in /kubernetes/pkg/controller/volume/attachdetach/populator/desired_state_of_world_populator.go
func (dswp *desiredStateOfWorldPopulator) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
	// Wait for the completion of a loop that started after sources are all ready, then set hasAddedPods accordingly
	klog.InfoS("Desired state populator starts to run")
	wait.PollUntil(dswp.loopSleepDuration, func() (bool, error) {
        //
		done := sourcesReady.AllReady()
		dswp.populatorLoop()
		return done, nil
	}, stopCh)
	dswp.hasAddedPodsLock.Lock()
	dswp.hasAddedPods = true
	dswp.hasAddedPodsLock.Unlock()
	wait.Until(dswp.populatorLoop, dswp.loopSleepDuration, stopCh)
}
```

desiredStateOfWorldPopulator通过populatorLoop()来更新DesiredStateOfWorld

```go
// in kubernetes/pkg/kubelet/volumemanager/populator/desired_state_of_world_populator.go
func (dswp *desiredStateOfWorldPopulator) populatorLoop() {
    dswp.findAndAddNewPods()

    // findAndRemoveDeletedPods() calls out to the container runtime to
    // determine if the containers for a given pod are terminated. This is
    // an expensive operation, therefore we limit the rate that
    // findAndRemoveDeletedPods() is called independently of the main
    // populator loop.
    if time.Since(dswp.timeOfLastGetPodStatus) < dswp.getPodStatusRetryDuration {
        klog.V(5).Infof(
            "Skipping findAndRemoveDeletedPods(). Not permitted until %v (getPodStatusRetryDuration %v).",
            dswp.timeOfLastGetPodStatus.Add(dswp.getPodStatusRetryDuration),
            dswp.getPodStatusRetryDuration)

        return
    }

    dswp.findAndRemoveDeletedPods()
}
```

findAndAddNewPods源码:
```go
//遍历Pod manager中所有pod
//过滤掉Terminated态的pod，进行processPodVolumes，把这些pod添加到desired state of world
//就是通过podManager获取所有的pods，然后调用processPodVolumes去更新desiredStateOfWorld。但是这样只能更新新增加的Pods的Volume信息。

// Iterate through all pods and add to desired state of world if they don't
// exist but should

func (dswp *desiredStateOfWorldPopulator) findAndAddNewPods() {
	// Map unique pod name to outer volume name to MountedVolume.
	mountedVolumesForPod := make(map[volumetypes.UniquePodName]map[string]cache.MountedVolume)
    //GetMountedVolumes返回一个要mountVolume的slice
	for _, mountedVolume := range dswp.actualStateOfWorld.GetMountedVolumes() {
        //返回Pod的UID
		mountedVolumes, exist := mountedVolumesForPod[mountedVolume.PodName]
		if !exist {
			mountedVolumes = make(map[string]cache.MountedVolume)
			mountedVolumesForPod[mountedVolume.PodName] = mountedVolumes
		}
        //添加Pod名字
		mountedVolumes[mountedVolume.OuterVolumeSpecName] = mountedVolume
	}
    //GetPods返回 []*v1.Pod
	for _, pod := range dswp.podManager.GetPods() {
		//过滤掉Terminated态的pod
        if dswp.podStateProvider.ShouldPodContainersBeTerminating(pod.UID) {
			// Do not (re)add volumes for pods that can't also be starting containers
			continue
		}
		dswp.processPodVolumes(pod, mountedVolumesForPod)
	}
}

//in kubernetes/pkg/kubelet/volumemanager/cache/actual_state_of_world.go


func (asw *actualStateOfWorld) GetMountedVolumes() []MountedVolume {
    //加锁？？
	asw.RLock()
	defer asw.RUnlock()
	mountedVolume := make([]MountedVolume, 0 /* len */, len(asw.attachedVolumes) /* cap */)
    //遍历attach的volume
    //attachedVolumes map[v1.UniqueVolumeName]attachedVolume
    //attachedVolume是一个volume对象
    //volumeObj是attachedVolume结构体
    //attachedVolume.mountedPods
    //mountedPods map[volumetypes.UniquePodName]mountedPod
    //podObj是mounedPod结构体
	for _, volumeObj := range asw.attachedVolumes {
		for _, podObj := range volumeObj.mountedPods {
            //operationexecutor.VolumeMounted代表Volume被成功mounted,可以更改asw
			if podObj.volumeMountStateForPod == operationexecutor.VolumeMounted {
				mountedVolume = append(
					mountedVolume,
					getMountedVolume(&podObj, &volumeObj))
			}
		}
	}
	return mountedVolume
}


type actualStateOfWorld struct {
	// nodeName is the name of this node. This value is passed to Attach/Detach
	nodeName types.NodeName

	// attachedVolumes is a map containing the set of volumes the kubelet volume
	// manager believes to be successfully attached to this node. Volume types
	// that do not implement an attacher interface are assumed to be in this
	// state by default.
	// The key in this map is the name of the volume and the value is an object
	// containing more information about the attached volume.
	attachedVolumes map[v1.UniqueVolumeName]attachedVolume

	// volumePluginMgr is the volume plugin manager used to create volume
	// plugin objects.
	volumePluginMgr *volume.VolumePluginMgr
	sync.RWMutex
}


type mountedPod struct {
	// the name of the pod
	podName volumetypes.UniquePodName

	// the UID of the pod
	podUID types.UID

	// mounter used to mount
	mounter volume.Mounter

	// mapper used to block volumes support
	blockVolumeMapper volume.BlockVolumeMapper

	// spec is the volume spec containing the specification for this volume.
	// Used to generate the volume plugin object, and passed to plugin methods.
	// In particular, the Unmount method uses spec.Name() as the volumeSpecName
	// in the mount path:
	// /var/lib/kubelet/pods/{podUID}/volumes/{escapeQualifiedPluginName}/{volumeSpecName}/
	volumeSpec *volume.Spec

	// outerVolumeSpecName is the volume.Spec.Name() of the volume as referenced
	// directly in the pod. If the volume was referenced through a persistent
	// volume claim, this contains the volume.Spec.Name() of the persistent
	// volume claim
	outerVolumeSpecName string

	// remountRequired indicates the underlying volume has been successfully
	// mounted to this pod but it should be remounted to reflect changes in the
	// referencing pod.
	// Atomically updating volumes depend on this to update the contents of the
	// volume. All volume mounting calls should be idempotent so a second mount
	// call for volumes that do not need to update contents should not fail.
	remountRequired bool

	// volumeGidValue contains the value of the GID annotation, if present.
	volumeGidValue string

	// volumeMountStateForPod stores state of volume mount for the pod. if it is:
	//   - VolumeMounted: means volume for pod has been successfully mounted
	//   - VolumeMountUncertain: means volume for pod may not be mounted, but it must be unmounted

    //VolumeMountState是string
	volumeMountStateForPod operationexecutor.VolumeMountState
}

// getMountedVolume constructs and returns a MountedVolume object from the given
// mountedPod and attachedVolume objects.
func getMountedVolume(
	mountedPod *mountedPod, attachedVolume *attachedVolume) MountedVolume {
	return MountedVolume{
		MountedVolume: operationexecutor.MountedVolume{
			PodName:             mountedPod.podName,
			VolumeName:          attachedVolume.volumeName,
			InnerVolumeSpecName: mountedPod.volumeSpec.Name(),
			OuterVolumeSpecName: mountedPod.outerVolumeSpecName,
			PluginName:          attachedVolume.pluginName,
			PodUID:              mountedPod.podUID,
			Mounter:             mountedPod.mounter,
			BlockVolumeMapper:   mountedPod.blockVolumeMapper,
			VolumeGidValue:      mountedPod.volumeGidValue,
			VolumeSpec:          mountedPod.volumeSpec,
			DeviceMountPath:     attachedVolume.deviceMountPath}}
}






//更新desiredStateOfWorld
// processPodVolumes processes the volumes in the given pod and adds them to the
// desired state of the world.
func (dswp *desiredStateOfWorldPopulator) processPodVolumes(
    pod *v1.Pod,
    mountedVolumesForPod map[volumetypes.UniquePodName]map[string]cache.MountedVolume,
    processedVolumesForFSResize sets.String) {
    if pod == nil {
        return
    }

    //  获得Pod.UID
    uniquePodName := util.GetUniquePodName(pod)
    // 如果先前在processedPods map中，表示无需处理，提前返回
    if dswp.podPreviouslyProcessed(uniquePodName) {
        return
    }

    allVolumesAdded := true
    // 获取 全部 容器的mount信息container.VolumeMounts
    // 对pod下所有container的volumeDevices与volumeMounts加入map中
    mounts, devices := util.GetPodVolumeNames(pod)

    expandInUsePV := utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes)
    // Process volume spec for each volume defined in pod
    for _, podVolume := range pod.Spec.Volumes {
        if !mounts.Has(podVolume.Name) && !devices.Has(podVolume.Name) {
            // Volume is not used in the pod, ignore it.
            // pod中定义了pod.Spec.Volumes[x].name，但是容器没有挂载使用，则忽略
            klog.V(4).Infof("Skipping unused volume %q for pod %q", podVolume.Name, format.Pod(pod))
            continue
        }
        // createVolumeSpec创建并返回一个可变的volume.Spec的对象。如果需要，它可通过PVC的间接引用以获得PV对象。当无法获取卷时返回报错
        pvc, volumeSpec, volumeGidValue, err :=
            dswp.createVolumeSpec(podVolume, pod, mounts, devices)
        if err != nil {
            klog.Errorf(
                "Error processing volume %q for pod %q: %v",
                podVolume.Name,
                format.Pod(pod),
                err)
            dswp.desiredStateOfWorld.AddErrorToPod(uniquePodName, err.Error())
            allVolumesAdded = false
            continue
        }

        // Add volume to desired state of world
        // 调用FindPluginBySpec函数根据volume.spec找到volume plugin
        //  isAttachableVolume函数，检查插件是否需要attach,不是所有的插件都需要实现AttachableVolumePlugin接口
        // 记录volume与pod之间的关系
        // 对pod name标记为已处理，actual_state_of_world标记重新挂载
        _, err = dswp.desiredStateOfWorld.AddPodToVolume(
            uniquePodName, pod, volumeSpec, podVolume.Name, volumeGidValue)
        if err != nil {
            klog.Errorf(
                "Failed to add volume %s (specName: %s) for pod %q to desiredStateOfWorld: %v",
                podVolume.Name,
                volumeSpec.Name(),
                uniquePodName,
                err)
            dswp.desiredStateOfWorld.AddErrorToPod(uniquePodName, err.Error())
            allVolumesAdded = false
        } else {
            klog.V(4).Infof(
                "Added volume %q (volSpec=%q) for pod %q to desired state.",
                podVolume.Name,
                volumeSpec.Name(),
                uniquePodName)
        }
        // 是否有卷容量调整操作, 实际上是比较 pvc.Status.Capacity 和 pvc.Spec.Capacity
        // pvc.Spec.Capacity > pvc.Status.Capacity时，进行扩容处理
        if expandInUsePV {
            dswp.checkVolumeFSResize(pod, podVolume, pvc, volumeSpec,
                uniquePodName, mountedVolumesForPod, processedVolumesForFSResize)
        }
    }

    // some of the volume additions may have failed, should not mark this pod as fully processed
    if allVolumesAdded {
        dswp.markPodProcessed(uniquePodName)
        // New pod has been synced. Re-mount all volumes that need it
        // (e.g. DownwardAPI)
        dswp.actualStateOfWorld.MarkRemountRequired(uniquePodName)
        // Remove any stored errors for the pod, everything went well in this processPodVolumes
        dswp.desiredStateOfWorld.PopPodErrors(uniquePodName)
    } else if dswp.podHasBeenSeenOnce(uniquePodName) {
        // For the Pod which has been processed at least once, even though some volumes
        // may not have been reprocessed successfully this round, we still mark it as processed to avoid
        // processing it at a very high frequency. The pod will be reprocessed when volume manager calls
        // ReprocessPod() which is triggered by SyncPod.
        dswp.markPodProcessed(uniquePodName)
    }

}

```


由于findAndRemoveDeletedPods 代价比较高昂，因此会检查执行的间隔时间。
遍历desiredStateOfWorld.GetVolumesToMount()的挂载volumes，根据volumeToMount.Pod判断该Volume所属的Pod是否存在于podManager。
如果存在podExists，则继续判断pod是否终止：如果pod为终止则忽略
根据containerRuntime进一步判断pod中的全部容器是否终止：如果该pod仍有容器未终止，则忽略
根据actualStateOfWorld.PodExistsInVolume判断：Actual state没有该pod的挂载volume，但pod manager仍有该pod，则忽略
删除管理器中该pod的该挂载卷：desiredStateOfWorld.DeletePodFromVolume(volumeToMount.PodName, volumeToMount.VolumeName)
删除管理器中该pod信息(desiredStateOfWorldPopulator.pods[volumeToMount.PodName])：deleteProcessedPod(volumeToMount.PodName)
简单说，对于pod manager已经不存在的pods，findAndRemoveDeletedPods会删除更新desiredStateOfWorld中这些pod和其volume记录
```go
// Iterate through all pods in desired state of world, and remove if they no
// longer exist
func (dswp *desiredStateOfWorldPopulator) findAndRemoveDeletedPods() {
    var runningPods []*kubecontainer.Pod

    runningPodsFetched := false
    for _, volumeToMount := range dswp.desiredStateOfWorld.GetVolumesToMount() {
        pod, podExists := dswp.podManager.GetPodByUID(volumeToMount.Pod.UID)
        if podExists {

            // check if the attachability has changed for this volume
            if volumeToMount.PluginIsAttachable {
                attachableVolumePlugin, err := dswp.volumePluginMgr.FindAttachablePluginBySpec(volumeToMount.VolumeSpec)
                // only this means the plugin is truly non-attachable
                if err == nil && attachableVolumePlugin == nil {
                    // It is not possible right now for a CSI plugin to be both attachable and non-deviceMountable
                    // So the uniqueVolumeName should remain the same after the attachability change
                    dswp.desiredStateOfWorld.MarkVolumeAttachability(volumeToMount.VolumeName, false)
                    klog.Infof("Volume %v changes from attachable to non-attachable.", volumeToMount.VolumeName)
                    continue
                }
            }

            // Skip running pods
            if !dswp.isPodTerminated(pod) {
                continue
            }
            if dswp.keepTerminatedPodVolumes {
                continue
            }
        }

        // Once a pod has been deleted from kubelet pod manager, do not delete
        // it immediately from volume manager. Instead, check the kubelet
        // containerRuntime to verify that all containers in the pod have been
        // terminated.
        if !runningPodsFetched {
            var getPodsErr error
            runningPods, getPodsErr = dswp.kubeContainerRuntime.GetPods(false)
            if getPodsErr != nil {
                klog.Errorf(
                    "kubeContainerRuntime.findAndRemoveDeletedPods returned error %v.",
                    getPodsErr)
                continue
            }

            runningPodsFetched = true
            dswp.timeOfLastGetPodStatus = time.Now()
        }

        runningContainers := false
        for _, runningPod := range runningPods {
            if runningPod.ID == volumeToMount.Pod.UID {
                if len(runningPod.Containers) > 0 {
                    runningContainers = true
                }

                break
            }
        }

        if runningContainers {
            klog.V(4).Infof(
                "Pod %q still has one or more containers in the non-exited state. Therefore, it will not be removed from desired state.",
                format.Pod(volumeToMount.Pod))
            continue
        }
        exists, _, _ := dswp.actualStateOfWorld.PodExistsInVolume(volumeToMount.PodName, volumeToMount.VolumeName)
        if !exists && podExists {
            klog.V(4).Infof(
                volumeToMount.GenerateMsgDetailed(fmt.Sprintf("Actual state has not yet has this volume mounted information and pod (%q) still exists in pod manager, skip removing volume from desired state",
                    format.Pod(volumeToMount.Pod)), ""))
            continue
        }
        klog.V(4).Infof(volumeToMount.GenerateMsgDetailed("Removing volume from desired state", ""))

        dswp.desiredStateOfWorld.DeletePodFromVolume(
            volumeToMount.PodName, volumeToMount.VolumeName)
        dswp.deleteProcessedPod(volumeToMount.PodName)
    }

    podsWithError := dswp.desiredStateOfWorld.GetPodsWithErrors()
    for _, podName := range podsWithError {
        if _, podExists := dswp.podManager.GetPodByUID(types.UID(podName)); !podExists {
            dswp.desiredStateOfWorld.PopPodErrors(podName)
        }
    }
}
//假如runningPodsFetched不存在，并不会立即马上删除卷信息记录。而是调用dswp.kubeContainerRuntime.GetPods(false)抓取Pod信息，这里是调用kubeContainerRuntime的GetPods函数。因此获取的都是runningPods信息，即正在运行的Pod信息。由于一个volume可以属于多个Pod，而一个Pod可以包含多个container，每个container都可以使用volume，所以他要扫描该volume所属的Pod的container信息，确保没有container使用该volume，才会删除该volume。

//desiredStateOfWorld就构建出来了，这是理想的volume状态，这里并没有发生实际的volume的创建删除挂载卸载操作。实际的操作由reconciler.Run(sourcesReady, stopCh)完成
```


reconciler 调谐器，即按desiredStateOfWorld来同步volume配置操作

主要流程
通过定时任务定期同步，reconcile就是一致性函数，保持desired和actual状态一致。

reconcile首先从actualStateOfWorld获取已经挂载的volume信息，然后查看该volume是否存在于desiredStateOfWorld,假如不存在就卸载。

接着从desiredStateOfWorld获取需要挂载的volumes。与actualStateOfWorld比较，假如没有挂载，则进行挂载。

这样存储就可以加载到主机attach，并挂载到容器目录mount。
```go
func (rc *reconciler) Run(stopCh <-chan struct{}) {
    wait.Until(rc.reconciliationLoopFunc(), rc.loopSleepDuration, stopCh)
}

//返回一个函数

func (rc *reconciler) reconciliationLoopFunc() func() {
    return func() {

        rc.reconcile()

        // Sync the state with the reality once after all existing pods are added to the desired state from all sources.
        // Otherwise, the reconstruct process may clean up pods' volumes that are still in use because
        // desired state of world does not contain a complete list of pods.
        if rc.populatorHasAddedPods() && !rc.StatesHasBeenSynced() {
            klog.Infof("Reconciler: start to sync state")
            rc.sync()
        }
    }
}

func (rc *reconciler) reconcile() {
	// Unmounts are triggered before mounts so that a volume that was
	// referenced by a pod that was deleted and is now referenced by another
	// pod is unmounted from the first pod before being mounted to the new
	// pod.
	rc.unmountVolumes()

	// Next we mount required volumes. This function could also trigger
	// attach if kubelet is responsible for attaching volumes.
	// If underlying PVC was resized while in-use then this function also handles volume
	// resizing.
	rc.mountOrAttachVolumes()

	// Ensure devices that should be detached/unmounted are detached/unmounted.
	rc.unmountDetachDevices()
}

func (rc *reconciler) unmountVolumes() {
    // Ensure volumes that should be unmounted are unmounted.
    for _, mountedVolume := range rc.actualStateOfWorld.GetAllMountedVolumes() {
        if !rc.desiredStateOfWorld.PodExistsInVolume(mountedVolume.PodName, mountedVolume.VolumeName) {
            // Volume is mounted, unmount it
            klog.V(5).Infof(mountedVolume.GenerateMsgDetailed("Starting operationExecutor.UnmountVolume", ""))
            // 此处UnmountVolume会根据具体的unmounter调用 CleanupMountPoint -> doCleanupMountPoint ，进行挂载卸载和目录删除
            // 这里可能会出现 对于挂载目录卸载失败的情况（社区有关孤儿pod的bug讨论），此时，kubelet的pod清理工作线程无法进行该挂载目录的直接删除
            err := rc.operationExecutor.UnmountVolume(
                mountedVolume.MountedVolume, rc.actualStateOfWorld, rc.kubeletPodsDir)
            if err != nil &&
                !nestedpendingoperations.IsAlreadyExists(err) &&
                !exponentialbackoff.IsExponentialBackoff(err) {
                // Ignore nestedpendingoperations.IsAlreadyExists and exponentialbackoff.IsExponentialBackoff errors, they are expected.
                // Log all other errors.
                klog.Errorf(mountedVolume.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.UnmountVolume failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error())
            }
            if err == nil {
                klog.Infof(mountedVolume.GenerateMsgDetailed("operationExecutor.UnmountVolume started", ""))
            }
        }
    }
}

func (rc *reconciler) mountOrAttachVolumes() {
	// Ensure volumes that should be attached/mounted are attached/mounted.
	for _, volumeToMount := range rc.desiredStateOfWorld.GetVolumesToMount() {
		volMounted, devicePath, err := rc.actualStateOfWorld.PodExistsInVolume(volumeToMount.PodName, volumeToMount.VolumeName, volumeToMount.PersistentVolumeSize)
		volumeToMount.DevicePath = devicePath
		if cache.IsVolumeNotAttachedError(err) {
			rc.waitForVolumeAttach(volumeToMount)
		} else if !volMounted || cache.IsRemountRequiredError(err) {
			rc.mountAttachedVolumes(volumeToMount, err)
		} else if cache.IsFSResizeRequiredError(err) {
			fsResizeRequiredErr, _ := err.(cache.FsResizeRequiredError)
			rc.expandVolume(volumeToMount, fsResizeRequiredErr.CurrentSize)
		}
	}
}

func (rc *reconciler) mountAttachedVolumes(volumeToMount cache.VolumeToMount, podExistError error) {
	// Volume is not mounted, or is already mounted, but requires remounting
	remountingLogStr := ""
	isRemount := cache.IsRemountRequiredError(podExistError)
	if isRemount {
		remountingLogStr = "Volume is already mounted to pod, but remount was requested."
	}
	klog.V(4).InfoS(volumeToMount.GenerateMsgDetailed("Starting operationExecutor.MountVolume", remountingLogStr), "pod", klog.KObj(volumeToMount.Pod))
	err := rc.operationExecutor.MountVolume(
		rc.waitForAttachTimeout,
		volumeToMount.VolumeToMount,
		rc.actualStateOfWorld,
		isRemount)
	if err != nil && !isExpectedError(err) {
		klog.ErrorS(err, volumeToMount.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.MountVolume failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error(), "pod", klog.KObj(volumeToMount.Pod))
	}
	if err == nil {
		if remountingLogStr == "" {
			klog.V(1).InfoS(volumeToMount.GenerateMsgDetailed("operationExecutor.MountVolume started", remountingLogStr), "pod", klog.KObj(volumeToMount.Pod))
		} else {
			klog.V(5).InfoS(volumeToMount.GenerateMsgDetailed("operationExecutor.MountVolume started", remountingLogStr), "pod", klog.KObj(volumeToMount.Pod))
		}
	}
}



func (oe *operationExecutor) MountVolume(
	waitForAttachTimeout time.Duration,
	volumeToMount VolumeToMount,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	isRemount bool) error {
	fsVolume, err := util.CheckVolumeModeFilesystem(volumeToMount.VolumeSpec)
	if err != nil {
		return err
	}
	var generatedOperations volumetypes.GeneratedOperations
	if fsVolume {
		// Filesystem volume case
		// Mount/remount a volume when a volume is attached
		generatedOperations = oe.operationGenerator.GenerateMountVolumeFunc(
			waitForAttachTimeout, volumeToMount, actualStateOfWorld, isRemount)

	} else {
		// Block volume case
		// Creates a map to device if a volume is attached
		generatedOperations, err = oe.operationGenerator.GenerateMapVolumeFunc(
			waitForAttachTimeout, volumeToMount, actualStateOfWorld)
	}
	if err != nil {
		return err
	}
	// Avoid executing mount/map from multiple pods referencing the
	// same volume in parallel
	podName := nestedpendingoperations.EmptyUniquePodName

	// TODO: remove this -- not necessary
	if !volumeToMount.PluginIsAttachable && !volumeToMount.PluginIsDeviceMountable {
		// volume plugins which are Non-attachable and Non-deviceMountable can execute mount for multiple pods
		// referencing the same volume in parallel
		podName = util.GetUniquePodName(volumeToMount.Pod)
	}

	// TODO mount_device
	return oe.pendingOperations.Run(
		volumeToMount.VolumeName, podName, "" /* nodeName */, generatedOperations)
}


func (og *operationGenerator) GenerateMountVolumeFunc(
	waitForAttachTimeout time.Duration,
	volumeToMount VolumeToMount,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	isRemount bool) volumetypes.GeneratedOperations {

	volumePluginName := unknownVolumePlugin
	volumePlugin, err :=
		og.volumePluginMgr.FindPluginBySpec(volumeToMount.VolumeSpec)
	if err == nil && volumePlugin != nil {
		volumePluginName = volumePlugin.GetPluginName()
	}

	mountVolumeFunc := func() volumetypes.OperationContext {
		//遍历所有plugin判断volumeSpec符合哪种plugin
        // Get mounter plugin
		volumePlugin, err := og.volumePluginMgr.FindPluginBySpec(volumeToMount.VolumeSpec)

		migrated := getMigratedStatusBySpec(volumeToMount.VolumeSpec)

		if err != nil || volumePlugin == nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.FindPluginBySpec failed", err)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		affinityErr := checkNodeAffinity(og, volumeToMount)
		if affinityErr != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.NodeAffinity check failed", affinityErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}
        //重要点
		volumeMounter, newMounterErr := volumePlugin.NewMounter(
			volumeToMount.VolumeSpec,
			volumeToMount.Pod,
			volume.VolumeOptions{})
		if newMounterErr != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.NewMounter initialization failed", newMounterErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		mountCheckError := checkMountOptionSupport(og, volumeToMount, volumePlugin)
		if mountCheckError != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MountOptionSupport check failed", mountCheckError)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		// Enforce ReadWriteOncePod access mode if it is the only one present. This is also enforced during scheduling.
		if utilfeature.DefaultFeatureGate.Enabled(features.ReadWriteOncePod) &&
			actualStateOfWorld.IsVolumeMountedElsewhere(volumeToMount.VolumeName, volumeToMount.PodName) &&
			// Because we do not know what access mode the pod intends to use if there are multiple.
			len(volumeToMount.VolumeSpec.PersistentVolume.Spec.AccessModes) == 1 &&
			v1helper.ContainsAccessMode(volumeToMount.VolumeSpec.PersistentVolume.Spec.AccessModes, v1.ReadWriteOncePod) {

			err = goerrors.New("volume uses the ReadWriteOncePod access mode and is already in use by another pod")
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.SetUp failed", err)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		// Get attacher, if possible
		attachableVolumePlugin, _ :=
			og.volumePluginMgr.FindAttachablePluginBySpec(volumeToMount.VolumeSpec)
		var volumeAttacher volume.Attacher
		if attachableVolumePlugin != nil {
			volumeAttacher, _ = attachableVolumePlugin.NewAttacher()
		}

		// get deviceMounter, if possible
		deviceMountableVolumePlugin, _ := og.volumePluginMgr.FindDeviceMountablePluginBySpec(volumeToMount.VolumeSpec)
		var volumeDeviceMounter volume.DeviceMounter
		if deviceMountableVolumePlugin != nil {
			volumeDeviceMounter, _ = deviceMountableVolumePlugin.NewDeviceMounter()
		}

		var fsGroup *int64
		var fsGroupChangePolicy *v1.PodFSGroupChangePolicy
		if podSc := volumeToMount.Pod.Spec.SecurityContext; podSc != nil {
			if podSc.FSGroup != nil {
				fsGroup = podSc.FSGroup
			}
			if podSc.FSGroupChangePolicy != nil {
				fsGroupChangePolicy = podSc.FSGroupChangePolicy
			}
		}

		devicePath := volumeToMount.DevicePath
		if volumeAttacher != nil {
			// Wait for attachable volumes to finish attaching
			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.WaitForAttach entering", fmt.Sprintf("DevicePath %q", volumeToMount.DevicePath)), "pod", klog.KObj(volumeToMount.Pod))

			devicePath, err = volumeAttacher.WaitForAttach(
				volumeToMount.VolumeSpec, devicePath, volumeToMount.Pod, waitForAttachTimeout)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.WaitForAttach failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.WaitForAttach succeeded", fmt.Sprintf("DevicePath %q", devicePath)), "pod", klog.KObj(volumeToMount.Pod))
		}

		var resizeError error
		resizeOptions := volume.NodeResizeOptions{
			DevicePath: devicePath,
		}

		if volumeDeviceMounter != nil && actualStateOfWorld.GetDeviceMountState(volumeToMount.VolumeName) != DeviceGloballyMounted {
			deviceMountPath, err :=
				volumeDeviceMounter.GetDeviceMountPath(volumeToMount.VolumeSpec)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.GetDeviceMountPath failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			// Mount device to global mount path
			err = volumeDeviceMounter.MountDevice(
				volumeToMount.VolumeSpec,
				devicePath,
				deviceMountPath,
				volume.DeviceMounterArgs{FsGroup: fsGroup},
			)
			if err != nil {
				og.checkForFailedMount(volumeToMount, err)
				og.markDeviceErrorState(volumeToMount, devicePath, deviceMountPath, err, actualStateOfWorld)
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MountDevice failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.MountDevice succeeded", fmt.Sprintf("device mount path %q", deviceMountPath)), "pod", klog.KObj(volumeToMount.Pod))

			// Update actual state of world to reflect volume is globally mounted
			markDeviceMountedErr := actualStateOfWorld.MarkDeviceAsMounted(
				volumeToMount.VolumeName, devicePath, deviceMountPath)
			if markDeviceMountedErr != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MarkDeviceAsMounted failed", markDeviceMountedErr)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}
			// set staging path for volume expansion
			resizeOptions.DeviceStagePath = deviceMountPath
		}

		// Execute mount
        // 执行mount操作
		mountErr := volumeMounter.SetUp(volume.MounterArgs{
			FsUser:              util.FsUserFrom(volumeToMount.Pod),
			FsGroup:             fsGroup,
			DesiredSize:         volumeToMount.DesiredSizeLimit,
			FSGroupChangePolicy: fsGroupChangePolicy,
		})
		// Update actual state of world
		markOpts := MarkVolumeOpts{
			PodName:             volumeToMount.PodName,
			PodUID:              volumeToMount.Pod.UID,
			VolumeName:          volumeToMount.VolumeName,
			Mounter:             volumeMounter,
			OuterVolumeSpecName: volumeToMount.OuterVolumeSpecName,
			VolumeGidVolume:     volumeToMount.VolumeGidValue,
			VolumeSpec:          volumeToMount.VolumeSpec,
			VolumeMountState:    VolumeMounted,
		}
		if mountErr != nil {
			og.checkForFailedMount(volumeToMount, mountErr)
			og.markVolumeErrorState(volumeToMount, markOpts, mountErr, actualStateOfWorld)
			// On failure, return error. Caller will log and retry.
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.SetUp failed", mountErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		detailedMsg := volumeToMount.GenerateMsgDetailed("MountVolume.SetUp succeeded", "")
		verbosity := klog.Level(1)
		if isRemount {
			verbosity = klog.Level(4)
		}
		klog.V(verbosity).InfoS(detailedMsg, "pod", klog.KObj(volumeToMount.Pod))
		resizeOptions.DeviceMountPath = volumeMounter.GetPath()

		_, resizeError = og.expandVolumeDuringMount(volumeToMount, actualStateOfWorld, resizeOptions)
		if resizeError != nil {
			klog.Errorf("MountVolume.NodeExpandVolume failed with %v", resizeError)
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.Setup failed while expanding volume", resizeError)
			// At this point, MountVolume.Setup already succeeded, we should add volume into actual state
			// so that reconciler can clean up volume when needed. However, volume resize failed,
			// we should not mark the volume as mounted to avoid pod starts using it.
			// Considering the above situations, we mark volume as uncertain here so that reconciler will tigger
			// volume tear down when pod is deleted, and also makes sure pod will not start using it.
			if err := actualStateOfWorld.MarkVolumeMountAsUncertain(markOpts); err != nil {
				klog.Errorf(volumeToMount.GenerateErrorDetailed("MountVolume.MarkVolumeMountAsUncertain failed", err).Error())
			}
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		// record total time it takes to mount a volume. This is end to end time that includes waiting for volume to attach, node to be update
		// plugin call to succeed
		mountRequestTime := volumeToMount.MountRequestTime
		totalTimeTaken := time.Since(mountRequestTime).Seconds()
		util.RecordOperationLatencyMetric(util.GetFullQualifiedPluginNameForVolume(volumePluginName, volumeToMount.VolumeSpec), "overall_volume_mount", totalTimeTaken)

		markVolMountedErr := actualStateOfWorld.MarkVolumeAsMounted(markOpts)
		if markVolMountedErr != nil {
			// On failure, return error. Caller will log and retry.
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MarkVolumeAsMounted failed", markVolMountedErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}
		return volumetypes.NewOperationContext(nil, nil, migrated)
	}

	eventRecorderFunc := func(err *error) {
		if *err != nil {
			og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeWarning, kevents.FailedMountVolume, (*err).Error())
		}
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "volume_mount",
		OperationFunc:     mountVolumeFunc,
		EventRecorderFunc: eventRecorderFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(volumePluginName, volumeToMount.VolumeSpec), "volume_mount"),
	}
}


func (plugin *nfsPlugin) NewMounter(spec *volume.Spec, pod *v1.Pod, _ volume.VolumeOptions) (volume.Mounter, error) {
	return plugin.newMounterInternal(spec, pod, plugin.host.GetMounter(plugin.GetPluginName()))
}

func (plugin *nfsPlugin) newMounterInternal(spec *volume.Spec, pod *v1.Pod, mounter mount.Interface) (volume.Mounter, error) {
	source, readOnly, err := getVolumeSource(spec)
	if err != nil {
		return nil, err
	}
	return &nfsMounter{
		nfs: &nfs{
			volName:         spec.Name(),
			mounter:         mounter,
			pod:             pod,
			plugin:          plugin,
			MetricsProvider: volume.NewMetricsStatFS(getPath(pod.UID, spec.Name(), plugin.host)),
		},
		server:       getServerFromSource(source),
		exportPath:   source.Path,
		readOnly:     readOnly,
		mountOptions: util.MountOptionFromSpec(spec),
	}, nil
}

```
```go
// SetUp attaches the disk and bind mounts to the volume path.
func (nfsMounter *nfsMounter) SetUp(mounterArgs volume.MounterArgs) error {
	return nfsMounter.SetUpAt(nfsMounter.GetPath(), mounterArgs)
}

func (nfsMounter *nfsMounter) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
	notMnt, err := mount.IsNotMountPoint(nfsMounter.mounter, dir)
	klog.V(4).Infof("NFS mount set up: %s %v %v", dir, !notMnt, err)
	if err != nil && !os.IsNotExist(err) {
		return err
	}
	if !notMnt {
		return nil
	}
	if err := os.MkdirAll(dir, 0750); err != nil {
		return err
	}
	source := fmt.Sprintf("%s:%s", nfsMounter.server, nfsMounter.exportPath)
	options := []string{}
	if nfsMounter.readOnly {
		options = append(options, "ro")
	}
	mountOptions := util.JoinMountOptions(nfsMounter.mountOptions, options)
	err = nfsMounter.mounter.MountSensitiveWithoutSystemd(source, dir, "nfs", mountOptions, nil)
	if err != nil {
		notMnt, mntErr := mount.IsNotMountPoint(nfsMounter.mounter, dir)
		if mntErr != nil {
			klog.Errorf("IsNotMountPoint check failed: %v", mntErr)
			return err
		}
		if !notMnt {
			if mntErr = nfsMounter.mounter.Unmount(dir); mntErr != nil {
				klog.Errorf("Failed to unmount: %v", mntErr)
				return err
			}
			notMnt, mntErr := mount.IsNotMountPoint(nfsMounter.mounter, dir)
			if mntErr != nil {
				klog.Errorf("IsNotMountPoint check failed: %v", mntErr)
				return err
			}
			if !notMnt {
				// This is very odd, we don't expect it.  We'll try again next sync loop.
				klog.Errorf("%s is still mounted, despite call to unmount().  Will try again next sync loop.", dir)
				return err
			}
		}
		os.Remove(dir)
		return err
	}
	return nil
}
```

```go
// MountSensitiveWithoutSystemd is the same as MountSensitive() but disable using systemd mount.
func (mounter *Mounter) MountSensitiveWithoutSystemd(source string, target string, fstype string, options []string, sensitiveOptions []string) error {
	return mounter.MountSensitiveWithoutSystemdWithMountFlags(source, target, fstype, options, sensitiveOptions, nil /* mountFlags */)
}

// MountSensitiveWithoutSystemdWithMountFlags is the same as MountSensitiveWithoutSystemd with additional mount flags.
func (mounter *Mounter) MountSensitiveWithoutSystemdWithMountFlags(source string, target string, fstype string, options []string, sensitiveOptions []string, mountFlags []string) error {
	mounterPath := ""
	bind, bindOpts, bindRemountOpts, bindRemountOptsSensitive := MakeBindOptsSensitive(options, sensitiveOptions)
	if bind {
		err := mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, bindOpts, bindRemountOptsSensitive, mountFlags, false)
		if err != nil {
			return err
		}
		return mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, bindRemountOpts, bindRemountOptsSensitive, mountFlags, false)
	}
	// The list of filesystems that require containerized mounter on GCI image cluster
	fsTypesNeedMounter := map[string]struct{}{
		"nfs":       {},
		"glusterfs": {},
		"ceph":      {},
		"cifs":      {},
	}
	if _, ok := fsTypesNeedMounter[fstype]; ok {
		mounterPath = mounter.mounterPath
	}
	return mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, options, sensitiveOptions, mountFlags, false)
}

func (mounter *Mounter) doMount(mounterPath string, mountCmd string, source string, target string, fstype string, options []string, sensitiveOptions []string, mountFlags []string, systemdMountRequired bool) error {
	mountArgs, mountArgsLogStr := MakeMountArgsSensitiveWithMountFlags(source, target, fstype, options, sensitiveOptions, mountFlags)
	if len(mounterPath) > 0 {
		mountArgs = append([]string{mountCmd}, mountArgs...)
		mountArgsLogStr = mountCmd + " " + mountArgsLogStr
		mountCmd = mounterPath
	}

	if mounter.withSystemd && systemdMountRequired {
		// Try to run mount via systemd-run --scope. This will escape the
		// service where kubelet runs and any fuse daemons will be started in a
		// specific scope. kubelet service than can be restarted without killing
		// these fuse daemons.
		//
		// Complete command line (when mounterPath is not used):
		// systemd-run --description=... --scope -- mount -t <type> <what> <where>
		//
		// Expected flow:
		// * systemd-run creates a transient scope (=~ cgroup) and executes its
		//   argument (/bin/mount) there.
		// * mount does its job, forks a fuse daemon if necessary and finishes.
		//   (systemd-run --scope finishes at this point, returning mount's exit
		//   code and stdout/stderr - thats one of --scope benefits).
		// * systemd keeps the fuse daemon running in the scope (i.e. in its own
		//   cgroup) until the fuse daemon dies (another --scope benefit).
		//   Kubelet service can be restarted and the fuse daemon survives.
		// * When the fuse daemon dies (e.g. during unmount) systemd removes the
		//   scope automatically.
		//
		// systemd-mount is not used because it's too new for older distros
		// (CentOS 7, Debian Jessie).
		mountCmd, mountArgs, mountArgsLogStr = AddSystemdScopeSensitive("systemd-run", target, mountCmd, mountArgs, mountArgsLogStr)
		// } else {
		// No systemd-run on the host (or we failed to check it), assume kubelet
		// does not run as a systemd service.
		// No code here, mountCmd and mountArgs are already populated.
	}

	// Logging with sensitive mount options removed.
	klog.V(4).Infof("Mounting cmd (%s) with arguments (%s)", mountCmd, mountArgsLogStr)
	command := exec.Command(mountCmd, mountArgs...)
	output, err := command.CombinedOutput()
	if err != nil {
		if err.Error() == errNoChildProcesses {
			if command.ProcessState.Success() {
				// We don't consider errNoChildProcesses an error if the process itself succeeded (see - k/k issue #103753).
				return nil
			}
			// Rewrite err with the actual exit error of the process.
			err = &exec.ExitError{ProcessState: command.ProcessState}
		}
		klog.Errorf("Mount failed: %v\nMounting command: %s\nMounting arguments: %s\nOutput: %s\n", err, mountCmd, mountArgsLogStr, string(output))
		return fmt.Errorf("mount failed: %v\nMounting command: %s\nMounting arguments: %s\nOutput: %s",
			err, mountCmd, mountArgsLogStr, string(output))
	}
	return err
}

func (c *Cmd) CombinedOutput() ([]byte, error) {
	if c.Stdout != nil {
		return nil, errors.New("exec: Stdout already set")
	}
	if c.Stderr != nil {
		return nil, errors.New("exec: Stderr already set")
	}
	var b bytes.Buffer
	c.Stdout = &b
	c.Stderr = &b
	err := c.Run()
	return b.Bytes(), err
}

```


NewPluginManager:

```go
//有点看不懂？？
func (grm *nestedPendingOperations) Run(
	volumeName v1.UniqueVolumeName,
	podName volumetypes.UniquePodName,
	nodeName types.NodeName,
	generatedOperations volumetypes.GeneratedOperations) error {
	grm.lock.Lock()
	defer grm.lock.Unlock()

	opKey := operationKey{volumeName, podName, nodeName}

	opExists, previousOpIndex := grm.isOperationExists(opKey)
	if opExists {
		previousOp := grm.operations[previousOpIndex]
		// Operation already exists
		if previousOp.operationPending {
			// Operation is pending
			return NewAlreadyExistsError(opKey)
		}

		backOffErr := previousOp.expBackoff.SafeToRetry(fmt.Sprintf("%+v", opKey))
		if backOffErr != nil {
			if previousOp.operationName == generatedOperations.OperationName {
				return backOffErr
			}
			// previous operation and new operation are different. reset op. name and exp. backoff
			grm.operations[previousOpIndex].operationName = generatedOperations.OperationName
			grm.operations[previousOpIndex].expBackoff = exponentialbackoff.ExponentialBackoff{}
		}

		// Update existing operation to mark as pending.
		grm.operations[previousOpIndex].operationPending = true
		grm.operations[previousOpIndex].key = opKey
	} else {
		// Create a new operation
		grm.operations = append(grm.operations,
			operation{
				key:              opKey,
				operationPending: true,
				operationName:    generatedOperations.OperationName,
				expBackoff:       exponentialbackoff.ExponentialBackoff{},
			})
	}

	go func() (eventErr, detailedErr error) {
		// Handle unhandled panics (very unlikely)
		defer k8sRuntime.HandleCrash()
		// Handle completion of and error, if any, from operationFunc()
		defer grm.operationComplete(opKey, &detailedErr)
		return generatedOperations.Run()
	}()

	return nil
}


//一些会涉及的结构体

// VolumePluginMgr tracks registered plugins.
type VolumePluginMgr struct {
	mutex                     sync.RWMutex
	plugins                   map[string]VolumePlugin
	prober                    DynamicPluginProber
	probedPlugins             map[string]VolumePlugin
	loggedDeprecationWarnings sets.String
	Host                      VolumeHost
}

// Spec is an internal representation of a volume.  All API volume types translate to Spec.
type Spec struct {
	Volume                          *v1.Volume
	PersistentVolume                *v1.PersistentVolume
	ReadOnly                        bool
	InlineVolumeSpecForCSIMigration bool
	Migrated                        bool
}

func (og *operationGenerator) GenerateMountVolumeFunc(
	waitForAttachTimeout time.Duration,
	volumeToMount VolumeToMount,
	actualStateOfWorld ActualStateOfWorldMounterUpdater,
	isRemount bool) volumetypes.GeneratedOperations {

	volumePluginName := unknownVolumePlugin
    //找到合适类型的volumePlugin
	volumePlugin, err :=
		og.volumePluginMgr.FindPluginBySpec(volumeToMount.VolumeSpec)
	if err == nil && volumePlugin != nil {
        //返回PluginName string
		volumePluginName = volumePlugin.GetPluginName()
	}

	mountVolumeFunc := func() volumetypes.OperationContext {
		// Get mounter plugin
		volumePlugin, err := og.volumePluginMgr.FindPluginBySpec(volumeToMount.VolumeSpec)

		migrated := getMigratedStatusBySpec(volumeToMount.VolumeSpec)

		if err != nil || volumePlugin == nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.FindPluginBySpec failed", err)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		affinityErr := checkNodeAffinity(og, volumeToMount)
		if affinityErr != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.NodeAffinity check failed", affinityErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}
        // 重点
		volumeMounter, newMounterErr := volumePlugin.NewMounter(
			volumeToMount.VolumeSpec,
			volumeToMount.Pod,
			volume.VolumeOptions{})
		if newMounterErr != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.NewMounter initialization failed", newMounterErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		mountCheckError := checkMountOptionSupport(og, volumeToMount, volumePlugin)
		if mountCheckError != nil {
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MountOptionSupport check failed", mountCheckError)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		// Enforce ReadWriteOncePod access mode if it is the only one present. This is also enforced during scheduling.
		if utilfeature.DefaultFeatureGate.Enabled(features.ReadWriteOncePod) &&
			actualStateOfWorld.IsVolumeMountedElsewhere(volumeToMount.VolumeName, volumeToMount.PodName) &&
			// Because we do not know what access mode the pod intends to use if there are multiple.
			len(volumeToMount.VolumeSpec.PersistentVolume.Spec.AccessModes) == 1 &&
			v1helper.ContainsAccessMode(volumeToMount.VolumeSpec.PersistentVolume.Spec.AccessModes, v1.ReadWriteOncePod) {

			err = goerrors.New("volume uses the ReadWriteOncePod access mode and is already in use by another pod")
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.SetUp failed", err)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		// Get attacher, if possible
		attachableVolumePlugin, _ :=
			og.volumePluginMgr.FindAttachablePluginBySpec(volumeToMount.VolumeSpec)
		var volumeAttacher volume.Attacher
		if attachableVolumePlugin != nil {
			volumeAttacher, _ = attachableVolumePlugin.NewAttacher()
		}

		// get deviceMounter, if possible
		deviceMountableVolumePlugin, _ := og.volumePluginMgr.FindDeviceMountablePluginBySpec(volumeToMount.VolumeSpec)
		var volumeDeviceMounter volume.DeviceMounter
		if deviceMountableVolumePlugin != nil {
			volumeDeviceMounter, _ = deviceMountableVolumePlugin.NewDeviceMounter()
		}

		var fsGroup *int64
		var fsGroupChangePolicy *v1.PodFSGroupChangePolicy
		if podSc := volumeToMount.Pod.Spec.SecurityContext; podSc != nil {
			if podSc.FSGroup != nil {
				fsGroup = podSc.FSGroup
			}
			if podSc.FSGroupChangePolicy != nil {
				fsGroupChangePolicy = podSc.FSGroupChangePolicy
			}
		}

		devicePath := volumeToMount.DevicePath
		if volumeAttacher != nil {
			// Wait for attachable volumes to finish attaching
			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.WaitForAttach entering", fmt.Sprintf("DevicePath %q", volumeToMount.DevicePath)), "pod", klog.KObj(volumeToMount.Pod))

			devicePath, err = volumeAttacher.WaitForAttach(
				volumeToMount.VolumeSpec, devicePath, volumeToMount.Pod, waitForAttachTimeout)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.WaitForAttach failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.WaitForAttach succeeded", fmt.Sprintf("DevicePath %q", devicePath)), "pod", klog.KObj(volumeToMount.Pod))
		}

		var resizeError error
		resizeOptions := volume.NodeResizeOptions{
			DevicePath: devicePath,
		}

		if volumeDeviceMounter != nil && actualStateOfWorld.GetDeviceMountState(volumeToMount.VolumeName) != DeviceGloballyMounted {
			deviceMountPath, err :=
				volumeDeviceMounter.GetDeviceMountPath(volumeToMount.VolumeSpec)
			if err != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.GetDeviceMountPath failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			// Mount device to global mount path
			err = volumeDeviceMounter.MountDevice(
				volumeToMount.VolumeSpec,
				devicePath,
				deviceMountPath,
				volume.DeviceMounterArgs{FsGroup: fsGroup},
			)
			if err != nil {
				og.checkForFailedMount(volumeToMount, err)
				og.markDeviceErrorState(volumeToMount, devicePath, deviceMountPath, err, actualStateOfWorld)
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MountDevice failed", err)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}

			klog.InfoS(volumeToMount.GenerateMsgDetailed("MountVolume.MountDevice succeeded", fmt.Sprintf("device mount path %q", deviceMountPath)), "pod", klog.KObj(volumeToMount.Pod))

			// Update actual state of world to reflect volume is globally mounted
			markDeviceMountedErr := actualStateOfWorld.MarkDeviceAsMounted(
				volumeToMount.VolumeName, devicePath, deviceMountPath)
			if markDeviceMountedErr != nil {
				// On failure, return error. Caller will log and retry.
				eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MarkDeviceAsMounted failed", markDeviceMountedErr)
				return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
			}
			// set staging path for volume expansion
			resizeOptions.DeviceStagePath = deviceMountPath
		}

		// Execute mount
		mountErr := volumeMounter.SetUp(volume.MounterArgs{
			FsUser:              util.FsUserFrom(volumeToMount.Pod),
			FsGroup:             fsGroup,
			DesiredSize:         volumeToMount.DesiredSizeLimit,
			FSGroupChangePolicy: fsGroupChangePolicy,
		})
		// Update actual state of world
		markOpts := MarkVolumeOpts{
			PodName:             volumeToMount.PodName,
			PodUID:              volumeToMount.Pod.UID,
			VolumeName:          volumeToMount.VolumeName,
			Mounter:             volumeMounter,
			OuterVolumeSpecName: volumeToMount.OuterVolumeSpecName,
			VolumeGidVolume:     volumeToMount.VolumeGidValue,
			VolumeSpec:          volumeToMount.VolumeSpec,
			VolumeMountState:    VolumeMounted,
		}
		if mountErr != nil {
			og.checkForFailedMount(volumeToMount, mountErr)
			og.markVolumeErrorState(volumeToMount, markOpts, mountErr, actualStateOfWorld)
			// On failure, return error. Caller will log and retry.
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.SetUp failed", mountErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		detailedMsg := volumeToMount.GenerateMsgDetailed("MountVolume.SetUp succeeded", "")
		verbosity := klog.Level(1)
		if isRemount {
			verbosity = klog.Level(4)
		}
		klog.V(verbosity).InfoS(detailedMsg, "pod", klog.KObj(volumeToMount.Pod))
		resizeOptions.DeviceMountPath = volumeMounter.GetPath()

		_, resizeError = og.expandVolumeDuringMount(volumeToMount, actualStateOfWorld, resizeOptions)
		if resizeError != nil {
			klog.Errorf("MountVolume.NodeExpandVolume failed with %v", resizeError)
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.Setup failed while expanding volume", resizeError)
			// At this point, MountVolume.Setup already succeeded, we should add volume into actual state
			// so that reconciler can clean up volume when needed. However, volume resize failed,
			// we should not mark the volume as mounted to avoid pod starts using it.
			// Considering the above situations, we mark volume as uncertain here so that reconciler will tigger
			// volume tear down when pod is deleted, and also makes sure pod will not start using it.
			if err := actualStateOfWorld.MarkVolumeMountAsUncertain(markOpts); err != nil {
				klog.Errorf(volumeToMount.GenerateErrorDetailed("MountVolume.MarkVolumeMountAsUncertain failed", err).Error())
			}
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}

		// record total time it takes to mount a volume. This is end to end time that includes waiting for volume to attach, node to be update
		// plugin call to succeed
		mountRequestTime := volumeToMount.MountRequestTime
		totalTimeTaken := time.Since(mountRequestTime).Seconds()
		util.RecordOperationLatencyMetric(util.GetFullQualifiedPluginNameForVolume(volumePluginName, volumeToMount.VolumeSpec), "overall_volume_mount", totalTimeTaken)

		markVolMountedErr := actualStateOfWorld.MarkVolumeAsMounted(markOpts)
		if markVolMountedErr != nil {
			// On failure, return error. Caller will log and retry.
			eventErr, detailedErr := volumeToMount.GenerateError("MountVolume.MarkVolumeAsMounted failed", markVolMountedErr)
			return volumetypes.NewOperationContext(eventErr, detailedErr, migrated)
		}
		return volumetypes.NewOperationContext(nil, nil, migrated)
	}

	eventRecorderFunc := func(err *error) {
		if *err != nil {
			og.recorder.Eventf(volumeToMount.Pod, v1.EventTypeWarning, kevents.FailedMountVolume, (*err).Error())
		}
	}

	return volumetypes.GeneratedOperations{
		OperationName:     "volume_mount",
		OperationFunc:     mountVolumeFunc,
		EventRecorderFunc: eventRecorderFunc,
		CompleteFunc:      util.OperationCompleteHook(util.GetFullQualifiedPluginNameForVolume(volumePluginName, volumeToMount.VolumeSpec), "volume_mount"),
	}
}

```




CleanupMountPoint -> doCleanupMountPoint
具体volume卸载操作

如果是挂载点，则先卸载mounter.Unmount(mountPath)
os.Remove(mountPath)

```go
// CleanupMountPoint unmounts the given path and deletes the remaining directory
// if successful. If extensiveMountPointCheck is true IsNotMountPoint will be
// called instead of IsLikelyNotMountPoint. IsNotMountPoint is more expensive
// but properly handles bind mounts within the same fs.
func CleanupMountPoint(mountPath string, mounter Interface, extensiveMountPointCheck bool) error {
    pathExists, pathErr := PathExists(mountPath)
    if !pathExists {
        klog.Warningf("Warning: Unmount skipped because path does not exist: %v", mountPath)
        return nil
    }
    corruptedMnt := IsCorruptedMnt(pathErr)
    if pathErr != nil && !corruptedMnt {
        return fmt.Errorf("Error checking path: %v", pathErr)
    }
    return doCleanupMountPoint(mountPath, mounter, extensiveMountPointCheck, corruptedMnt)
}

// doCleanupMountPoint unmounts the given path and
// deletes the remaining directory if successful.
// if extensiveMountPointCheck is true
// IsNotMountPoint will be called instead of IsLikelyNotMountPoint.
// IsNotMountPoint is more expensive but properly handles bind mounts within the same fs.
// if corruptedMnt is true, it means that the mountPath is a corrupted mountpoint, and the mount point check
// will be skipped
func doCleanupMountPoint(mountPath string, mounter Interface, extensiveMountPointCheck bool, corruptedMnt bool) error {
    var notMnt bool
    var err error

    //IsNotMountPoint和IsLikelyNotMountPoint作用？？
    if !corruptedMnt {
        if extensiveMountPointCheck {
            notMnt, err = IsNotMountPoint(mounter, mountPath)
        } else {
            notMnt, err = mounter.IsLikelyNotMountPoint(mountPath)
        }

        if err != nil {
            return err
        }

        if notMnt {
            klog.Warningf("Warning: %q is not a mountpoint, deleting", mountPath)
            return os.Remove(mountPath)
        }
    }

    // Unmount the mount path
    klog.V(4).Infof("%q is a mountpoint, unmounting", mountPath)
    if err := mounter.Unmount(mountPath); err != nil {
        return err
    }

    if extensiveMountPointCheck {
        notMnt, err = IsNotMountPoint(mounter, mountPath)
    } else {
        notMnt, err = mounter.IsLikelyNotMountPoint(mountPath)
    }
    if err != nil {
        return err
    }
    if notMnt {
        klog.V(4).Infof("%q is unmounted, deleting the directory", mountPath)
        return os.Remove(mountPath)
    }
    return fmt.Errorf("Failed to unmount path %v", mountPath)
}

```

## 源码分析
in /pkg/apis/storage/types.go
描述storge有关的结构体类型
疑问1：为什么定义完接口还要定义结构体？？go中的接口与chan



```go

// podStateProvider can determine if a pod is going to be terminated
type podStateProvider interface {
	ShouldPodContainersBeTerminating(k8stypes.UID) bool
	ShouldPodRuntimeBeRemoved(k8stypes.UID) bool
}
```

```go
// NewVolumeManager returns a new concrete instance implementing the
// VolumeManager interface.
//
// kubeClient - kubeClient is the kube API client used by DesiredStateOfWorldPopulator
//   to communicate with the API server to fetch PV and PVC objects
// volumePluginMgr - the volume plugin manager used to access volume plugins.
//   Must be pre-initialized.
func NewVolumeManager(
	controllerAttachDetachEnabled bool,
	nodeName k8stypes.NodeName,
	podManager pod.Manager,
	podStateProvider podStateProvider,
	kubeClient clientset.Interface,
	volumePluginMgr *volume.VolumePluginMgr,
	kubeContainerRuntime container.Runtime,
	mounter mount.Interface,
	hostutil hostutil.HostUtils,
	kubeletPodsDir string,
	recorder record.EventRecorder,
	keepTerminatedPodVolumes bool,
	blockVolumePathHandler volumepathhandler.BlockVolumePathHandler) VolumeManager {

	vm := &volumeManager{
		kubeClient:          kubeClient,
		volumePluginMgr:     volumePluginMgr,
		desiredStateOfWorld: cache.NewDesiredStateOfWorld(volumePluginMgr),
		actualStateOfWorld:  cache.NewActualStateOfWorld(nodeName, volumePluginMgr),
		operationExecutor: operationexecutor.NewOperationExecutor(operationexecutor.NewOperationGenerator(
			kubeClient,
			volumePluginMgr,
			recorder,
			blockVolumePathHandler)),
	}

	intreeToCSITranslator := csitrans.New()
	csiMigratedPluginManager := csimigration.NewPluginManager(intreeToCSITranslator, utilfeature.DefaultFeatureGate)

	vm.intreeToCSITranslator = intreeToCSITranslator
	vm.csiMigratedPluginManager = csiMigratedPluginManager
	vm.desiredStateOfWorldPopulator = populator.NewDesiredStateOfWorldPopulator(
		kubeClient,
		desiredStateOfWorldPopulatorLoopSleepPeriod,
		desiredStateOfWorldPopulatorGetPodStatusRetryDuration,
		podManager,
		podStateProvider,
		vm.desiredStateOfWorld,
		vm.actualStateOfWorld,
		kubeContainerRuntime,
		keepTerminatedPodVolumes,
		csiMigratedPluginManager,
		intreeToCSITranslator,
		volumePluginMgr)
	vm.reconciler = reconciler.NewReconciler(
		kubeClient,
		controllerAttachDetachEnabled,
		reconcilerLoopSleepPeriod,
		waitForAttachTimeout,
		nodeName,
		vm.desiredStateOfWorld,
		vm.actualStateOfWorld,
		vm.desiredStateOfWorldPopulator.HasAddedPods,
		vm.operationExecutor,
		mounter,
		hostutil,
		volumePluginMgr,
		kubeletPodsDir)

	return vm
}
```

```go
// volumeManager implements the VolumeManager interface
type volumeManager struct {
	// kubeClient is the kube API client used by DesiredStateOfWorldPopulator to
	// communicate with the API server to fetch PV and PVC objects
	kubeClient clientset.Interface

	// volumePluginMgr is the volume plugin manager used to access volume
	// plugins. It must be pre-initialized.
	volumePluginMgr *volume.VolumePluginMgr

	// desiredStateOfWorld is a data structure containing the desired state of
	// the world according to the volume manager: i.e. what volumes should be
	// attached and which pods are referencing the volumes).
	// The data structure is populated by the desired state of the world
	// populator using the kubelet pod manager.
	desiredStateOfWorld cache.DesiredStateOfWorld

	// actualStateOfWorld is a data structure containing the actual state of
	// the world according to the manager: i.e. which volumes are attached to
	// this node and what pods the volumes are mounted to.
	// The data structure is populated upon successful completion of attach,
	// detach, mount, and unmount actions triggered by the reconciler.
	actualStateOfWorld cache.ActualStateOfWorld

	// operationExecutor is used to start asynchronous attach, detach, mount,
	// and unmount operations.
	operationExecutor operationexecutor.OperationExecutor

	// reconciler runs an asynchronous periodic loop to reconcile the
	// desiredStateOfWorld with the actualStateOfWorld by triggering attach,
	// detach, mount, and unmount operations using the operationExecutor.
	reconciler reconciler.Reconciler

	// desiredStateOfWorldPopulator runs an asynchronous periodic loop to
	// populate the desiredStateOfWorld using the kubelet PodManager.
	desiredStateOfWorldPopulator populator.DesiredStateOfWorldPopulator

	// csiMigratedPluginManager keeps track of CSI migration status of plugins
	csiMigratedPluginManager csimigration.PluginManager

	// intreeToCSITranslator translates in-tree volume specs to CSI
	intreeToCSITranslator csimigration.InTreeToCSITranslator
}
```


in /pkg/kubelet/kubelet_pods.go