Container 
```go
type Container struct {
	// Name of the container specified as a DNS_LABEL.
	// Each container in a pod must have a unique name (DNS_LABEL).
	// Cannot be updated.
	Name string `json:"name" protobuf:"bytes,1,opt,name=name"`
	// Container image name.
	// More info: https://kubernetes.io/docs/concepts/containers/images
	// This field is optional to allow higher level config management to default or override
	// container images in workload controllers like Deployments and StatefulSets.
	// +optional
	Image string `json:"image,omitempty" protobuf:"bytes,2,opt,name=image"`
	// Entrypoint array. Not executed within a shell.
	// The container image's ENTRYPOINT is used if this is not provided.
	// Variable references $(VAR_NAME) are expanded using the container's environment. If a variable
	// cannot be resolved, the reference in the input string will be unchanged. Double $$ are reduced
	// to a single $, which allows for escaping the $(VAR_NAME) syntax: i.e. "$$(VAR_NAME)" will
	// produce the string literal "$(VAR_NAME)". Escaped references will never be expanded, regardless
	// of whether the variable exists or not. Cannot be updated.
	// More info: https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#running-a-command-in-a-shell
	// +optional
	Command []string `json:"command,omitempty" protobuf:"bytes,3,rep,name=command"`
	// Arguments to the entrypoint.
	// The container image's CMD is used if this is not provided.
	// Variable references $(VAR_NAME) are expanded using the container's environment. If a variable
	// cannot be resolved, the reference in the input string will be unchanged. Double $$ are reduced
	// to a single $, which allows for escaping the $(VAR_NAME) syntax: i.e. "$$(VAR_NAME)" will
	// produce the string literal "$(VAR_NAME)". Escaped references will never be expanded, regardless
	// of whether the variable exists or not. Cannot be updated.
	// More info: https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#running-a-command-in-a-shell
	// +optional
	Args []string `json:"args,omitempty" protobuf:"bytes,4,rep,name=args"`
	// Container's working directory.
	// If not specified, the container runtime's default will be used, which
	// might be configured in the container image.
	// Cannot be updated.
	// +optional
	WorkingDir string `json:"workingDir,omitempty" protobuf:"bytes,5,opt,name=workingDir"`
	// List of ports to expose from the container. Exposing a port here gives
	// the system additional information about the network connections a
	// container uses, but is primarily informational. Not specifying a port here
	// DOES NOT prevent that port from being exposed. Any port which is
	// listening on the default "0.0.0.0" address inside a container will be
	// accessible from the network.
	// Cannot be updated.
	// +optional
	// +patchMergeKey=containerPort
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=containerPort
	// +listMapKey=protocol
	Ports []ContainerPort `json:"ports,omitempty" patchStrategy:"merge" patchMergeKey:"containerPort" protobuf:"bytes,6,rep,name=ports"`
	// List of sources to populate environment variables in the container.
	// The keys defined within a source must be a C_IDENTIFIER. All invalid keys
	// will be reported as an event when the container is starting. When a key exists in multiple
	// sources, the value associated with the last source will take precedence.
	// Values defined by an Env with a duplicate key will take precedence.
	// Cannot be updated.
	// +optional
	EnvFrom []EnvFromSource `json:"envFrom,omitempty" protobuf:"bytes,19,rep,name=envFrom"`
	// List of environment variables to set in the container.
	// Cannot be updated.
	// +optional
	// +patchMergeKey=name
	// +patchStrategy=merge
	Env []EnvVar `json:"env,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,7,rep,name=env"`
	// Compute Resources required by this container.
	// Cannot be updated.
	// More info: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
	// +optional
	Resources ResourceRequirements `json:"resources,omitempty" protobuf:"bytes,8,opt,name=resources"`
	// Pod volumes to mount into the container's filesystem.
	// Cannot be updated.
	// +optional
	// +patchMergeKey=mountPath
	// +patchStrategy=merge
	VolumeMounts []VolumeMount `json:"volumeMounts,omitempty" patchStrategy:"merge" patchMergeKey:"mountPath" protobuf:"bytes,9,rep,name=volumeMounts"`
	// volumeDevices is the list of block devices to be used by the container.
	// +patchMergeKey=devicePath
	// +patchStrategy=merge
	// +optional
	VolumeDevices []VolumeDevice `json:"volumeDevices,omitempty" patchStrategy:"merge" patchMergeKey:"devicePath" protobuf:"bytes,21,rep,name=volumeDevices"`
	// Periodic probe of container liveness.
	// Container will be restarted if the probe fails.
	// Cannot be updated.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes
	// +optional
	LivenessProbe *Probe `json:"livenessProbe,omitempty" protobuf:"bytes,10,opt,name=livenessProbe"`
	// Periodic probe of container service readiness.
	// Container will be removed from service endpoints if the probe fails.
	// Cannot be updated.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes
	// +optional
	ReadinessProbe *Probe `json:"readinessProbe,omitempty" protobuf:"bytes,11,opt,name=readinessProbe"`
	// StartupProbe indicates that the Pod has successfully initialized.
	// If specified, no other probes are executed until this completes successfully.
	// If this probe fails, the Pod will be restarted, just as if the livenessProbe failed.
	// This can be used to provide different probe parameters at the beginning of a Pod's lifecycle,
	// when it might take a long time to load data or warm a cache, than during steady-state operation.
	// This cannot be updated.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes
	// +optional
	StartupProbe *Probe `json:"startupProbe,omitempty" protobuf:"bytes,22,opt,name=startupProbe"`
	// Actions that the management system should take in response to container lifecycle events.
	// Cannot be updated.
	// +optional
	Lifecycle *Lifecycle `json:"lifecycle,omitempty" protobuf:"bytes,12,opt,name=lifecycle"`
	// Optional: Path at which the file to which the container's termination message
	// will be written is mounted into the container's filesystem.
	// Message written is intended to be brief final status, such as an assertion failure message.
	// Will be truncated by the node if greater than 4096 bytes. The total message length across
	// all containers will be limited to 12kb.
	// Defaults to /dev/termination-log.
	// Cannot be updated.
	// +optional
	TerminationMessagePath string `json:"terminationMessagePath,omitempty" protobuf:"bytes,13,opt,name=terminationMessagePath"`
	// Indicate how the termination message should be populated. File will use the contents of
	// terminationMessagePath to populate the container status message on both success and failure.
	// FallbackToLogsOnError will use the last chunk of container log output if the termination
	// message file is empty and the container exited with an error.
	// The log output is limited to 2048 bytes or 80 lines, whichever is smaller.
	// Defaults to File.
	// Cannot be updated.
	// +optional
	TerminationMessagePolicy TerminationMessagePolicy `` /* 129-byte string literal not displayed */
	// Image pull policy.
	// One of Always, Never, IfNotPresent.
	// Defaults to Always if :latest tag is specified, or IfNotPresent otherwise.
	// Cannot be updated.
	// More info: https://kubernetes.io/docs/concepts/containers/images#updating-images
	// +optional
	ImagePullPolicy PullPolicy `json:"imagePullPolicy,omitempty" protobuf:"bytes,14,opt,name=imagePullPolicy,casttype=PullPolicy"`
	// SecurityContext defines the security options the container should be run with.
	// If set, the fields of SecurityContext override the equivalent fields of PodSecurityContext.
	// More info: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
	// +optional
	SecurityContext *SecurityContext `json:"securityContext,omitempty" protobuf:"bytes,15,opt,name=securityContext"`

	// Whether this container should allocate a buffer for stdin in the container runtime. If this
	// is not set, reads from stdin in the container will always result in EOF.
	// Default is false.
	// +optional
	Stdin bool `json:"stdin,omitempty" protobuf:"varint,16,opt,name=stdin"`
	// Whether the container runtime should close the stdin channel after it has been opened by
	// a single attach. When stdin is true the stdin stream will remain open across multiple attach
	// sessions. If stdinOnce is set to true, stdin is opened on container start, is empty until the
	// first client attaches to stdin, and then remains open and accepts data until the client disconnects,
	// at which time stdin is closed and remains closed until the container is restarted. If this
	// flag is false, a container processes that reads from stdin will never receive an EOF.
	// Default is false
	// +optional
	StdinOnce bool `json:"stdinOnce,omitempty" protobuf:"varint,17,opt,name=stdinOnce"`
	// Whether this container should allocate a TTY for itself, also requires 'stdin' to be true.
	// Default is false.
	// +optional
	TTY bool `json:"tty,omitempty" protobuf:"varint,18,opt,name=tty"`
}
```


VolumeMount
```go
type VolumeMount struct {
	// This must match the Name of a Volume.
	Name string `json:"name" protobuf:"bytes,1,opt,name=name"`
	// Mounted read-only if true, read-write otherwise (false or unspecified).
	// Defaults to false.
	// +optional
	ReadOnly bool `json:"readOnly,omitempty" protobuf:"varint,2,opt,name=readOnly"`
	// Path within the container at which the volume should be mounted.  Must
	// not contain ':'.
	MountPath string `json:"mountPath" protobuf:"bytes,3,opt,name=mountPath"`
	// Path within the volume from which the container's volume should be mounted.
	// Defaults to "" (volume's root).
	// +optional
	SubPath string `json:"subPath,omitempty" protobuf:"bytes,4,opt,name=subPath"`
	// mountPropagation determines how mounts are propagated from the host
	// to container and the other way around.
	// When not set, MountPropagationNone is used.
	// This field is beta in 1.10.
	// +optional
	MountPropagation *MountPropagationMode `json:"mountPropagation,omitempty" protobuf:"bytes,5,opt,name=mountPropagation,casttype=MountPropagationMode"`
	// Expanded path within the volume from which the container's volume should be mounted.
	// Behaves similarly to SubPath but environment variable references $(VAR_NAME) are expanded using the container's environment.
	// Defaults to "" (volume's root).
	// SubPathExpr and SubPath are mutually exclusive.
	// +optional
	SubPathExpr string `json:"subPathExpr,omitempty" protobuf:"bytes,6,opt,name=subPathExpr"`
}
```


VolumeToMount
VolumeToMount represents a volume that should be attached to this node and mounted to the PodName.
```go
type VolumeToMount struct {
    // VolumeName is the unique identifier for the volume that should be
    // mounted.
    // string
    VolumeName v1.UniqueVolumeName

    // PodName is the unique identifier for the pod that the volume should be
    // mounted to after it is attached.
    PodName volumetypes.UniquePodName

    // VolumeSpec is a volume spec containing the specification for the volume
    // that should be mounted. Used to create NewMounter. Used to generate
    // InnerVolumeSpecName.
    VolumeSpec *volume.Spec

    // outerVolumeSpecName is the podSpec.Volume[x].Name of the volume. If the
    // volume was referenced through a persistent volume claim, this contains
    // the podSpec.Volume[x].Name of the persistent volume claim.
    OuterVolumeSpecName string

    // Pod to mount the volume to. Used to create NewMounter.
    Pod *v1.Pod

    // PluginIsAttachable indicates that the plugin for this volume implements
    // the volume.Attacher interface
    PluginIsAttachable bool

    // PluginIsDeviceMountable indicates that the plugin for this volume implements
    // the volume.DeviceMounter interface
    PluginIsDeviceMountable bool

    // VolumeGidValue contains the value of the GID annotation, if present.
    VolumeGidValue string

    // DevicePath contains the path on the node where the volume is attached.
    // For non-attachable volumes this is empty.
    DevicePath string

    // ReportedInUse indicates that the volume was successfully added to the
    // VolumesInUse field in the node's status.
    ReportedInUse bool

    // DesiredSizeLimit indicates the desired upper bound on the size of the volume
    // (if so implemented)
    DesiredSizeLimit *resource.Quantity

    // time at which volume was requested to be mounted
    MountRequestTime time.Time

    // PersistentVolumeSize stores desired size of the volume.
    // usually this is the size if pv.Spec.Capacity
    PersistentVolumeSize resource.Quantity
}
```

Mount Mount specifies a host volume to mount into a container.
```go
type Mount struct {
    Name string
	// Path of the mount within the container.
	ContainerPath string `protobuf:"bytes,1,opt,name=container_path,json=containerPath,proto3" json:"container_path,omitempty"`
	// Path of the mount on the host. If the hostPath doesn't exist, then runtimes
	// should report error. If the hostpath is a symbolic link, runtimes should
	// follow the symlink and mount the real destination to container.
	HostPath string `protobuf:"bytes,2,opt,name=host_path,json=hostPath,proto3" json:"host_path,omitempty"`
	// If set, the mount is read-only.
	Readonly bool `protobuf:"varint,3,opt,name=readonly,proto3" json:"readonly,omitempty"`
	// If set, the mount needs SELinux relabeling.
	SelinuxRelabel bool `protobuf:"varint,4,opt,name=selinux_relabel,json=selinuxRelabel,proto3" json:"selinux_relabel,omitempty"`
	// Requested propagation mode.
	Propagation          MountPropagation `protobuf:"varint,5,opt,name=propagation,proto3,enum=runtime.v1.MountPropagation" json:"propagation,omitempty"`
	XXX_NoUnkeyedLiteral struct{}         `json:"-"`
	XXX_sizecache        int32            `json:"-"`
}
```


VolumeSource
```go
// Represents the source of a volume to mount. Only one of its members may be specified.
// 代表Volume类型只能选一种
// yaml文件文件中:volumes: name VolumeSource两个key
type VolumeSource struct {
	// hostPath represents a pre-existing file or directory on the host
	// machine that is directly exposed to the container. This is generally
	// used for system agents or other privileged things that are allowed
	// to see the host machine. Most containers will NOT need this.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#hostpath
	// ---
	// TODO(jonesdl) We need to restrict who can use host directory mounts and who can/can not
	// mount host directories as read/write.
	// +optional
	HostPath *HostPathVolumeSource `json:"hostPath,omitempty" protobuf:"bytes,1,opt,name=hostPath"`
	// emptyDir represents a temporary directory that shares a pod's lifetime.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#emptydir
	// +optional
	EmptyDir *EmptyDirVolumeSource `json:"emptyDir,omitempty" protobuf:"bytes,2,opt,name=emptyDir"`
	// gcePersistentDisk represents a GCE Disk resource that is attached to a
	// kubelet's host machine and then exposed to the pod.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#gcepersistentdisk
	// +optional
	GCEPersistentDisk *GCEPersistentDiskVolumeSource `json:"gcePersistentDisk,omitempty" protobuf:"bytes,3,opt,name=gcePersistentDisk"`
	// awsElasticBlockStore represents an AWS Disk resource that is attached to a
	// kubelet's host machine and then exposed to the pod.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#awselasticblockstore
	// +optional
	AWSElasticBlockStore *AWSElasticBlockStoreVolumeSource `json:"awsElasticBlockStore,omitempty" protobuf:"bytes,4,opt,name=awsElasticBlockStore"`
	// gitRepo represents a git repository at a particular revision.
	// DEPRECATED: GitRepo is deprecated. To provision a container with a git repo, mount an
	// EmptyDir into an InitContainer that clones the repo using git, then mount the EmptyDir
	// into the Pod's container.
	// +optional
	GitRepo *GitRepoVolumeSource `json:"gitRepo,omitempty" protobuf:"bytes,5,opt,name=gitRepo"`
	// secret represents a secret that should populate this volume.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#secret
	// +optional
	Secret *SecretVolumeSource `json:"secret,omitempty" protobuf:"bytes,6,opt,name=secret"`
	// nfs represents an NFS mount on the host that shares a pod's lifetime
	// More info: https://kubernetes.io/docs/concepts/storage/volumes#nfs
	// +optional
	NFS *NFSVolumeSource `json:"nfs,omitempty" protobuf:"bytes,7,opt,name=nfs"`
	// iscsi represents an ISCSI Disk resource that is attached to a
	// kubelet's host machine and then exposed to the pod.
	// More info: https://examples.k8s.io/volumes/iscsi/README.md
	// +optional
	ISCSI *ISCSIVolumeSource `json:"iscsi,omitempty" protobuf:"bytes,8,opt,name=iscsi"`
	// glusterfs represents a Glusterfs mount on the host that shares a pod's lifetime.
	// More info: https://examples.k8s.io/volumes/glusterfs/README.md
	// +optional
	Glusterfs *GlusterfsVolumeSource `json:"glusterfs,omitempty" protobuf:"bytes,9,opt,name=glusterfs"`
	// persistentVolumeClaimVolumeSource represents a reference to a
	// PersistentVolumeClaim in the same namespace.
	// More info: https://kubernetes.io/docs/concepts/storage/persistent-volumes#persistentvolumeclaims
	// +optional
	PersistentVolumeClaim *PersistentVolumeClaimVolumeSource `json:"persistentVolumeClaim,omitempty" protobuf:"bytes,10,opt,name=persistentVolumeClaim"`
	// rbd represents a Rados Block Device mount on the host that shares a pod's lifetime.
	// More info: https://examples.k8s.io/volumes/rbd/README.md
	// +optional
	RBD *RBDVolumeSource `json:"rbd,omitempty" protobuf:"bytes,11,opt,name=rbd"`
	// flexVolume represents a generic volume resource that is
	// provisioned/attached using an exec based plugin.
	// +optional
	FlexVolume *FlexVolumeSource `json:"flexVolume,omitempty" protobuf:"bytes,12,opt,name=flexVolume"`
	// cinder represents a cinder volume attached and mounted on kubelets host machine.
	// More info: https://examples.k8s.io/mysql-cinder-pd/README.md
	// +optional
	Cinder *CinderVolumeSource `json:"cinder,omitempty" protobuf:"bytes,13,opt,name=cinder"`
	// cephFS represents a Ceph FS mount on the host that shares a pod's lifetime
	// +optional
	CephFS *CephFSVolumeSource `json:"cephfs,omitempty" protobuf:"bytes,14,opt,name=cephfs"`
	// flocker represents a Flocker volume attached to a kubelet's host machine. This depends on the Flocker control service being running
	// +optional
	Flocker *FlockerVolumeSource `json:"flocker,omitempty" protobuf:"bytes,15,opt,name=flocker"`
	// downwardAPI represents downward API about the pod that should populate this volume
	// +optional
	DownwardAPI *DownwardAPIVolumeSource `json:"downwardAPI,omitempty" protobuf:"bytes,16,opt,name=downwardAPI"`
	// fc represents a Fibre Channel resource that is attached to a kubelet's host machine and then exposed to the pod.
	// +optional
	FC *FCVolumeSource `json:"fc,omitempty" protobuf:"bytes,17,opt,name=fc"`
	// azureFile represents an Azure File Service mount on the host and bind mount to the pod.
	// +optional
	AzureFile *AzureFileVolumeSource `json:"azureFile,omitempty" protobuf:"bytes,18,opt,name=azureFile"`
	// configMap represents a configMap that should populate this volume
	// +optional
	ConfigMap *ConfigMapVolumeSource `json:"configMap,omitempty" protobuf:"bytes,19,opt,name=configMap"`
	// vsphereVolume represents a vSphere volume attached and mounted on kubelets host machine
	// +optional
	VsphereVolume *VsphereVirtualDiskVolumeSource `json:"vsphereVolume,omitempty" protobuf:"bytes,20,opt,name=vsphereVolume"`
	// quobyte represents a Quobyte mount on the host that shares a pod's lifetime
	// +optional
	Quobyte *QuobyteVolumeSource `json:"quobyte,omitempty" protobuf:"bytes,21,opt,name=quobyte"`
	// azureDisk represents an Azure Data Disk mount on the host and bind mount to the pod.
	// +optional
	AzureDisk *AzureDiskVolumeSource `json:"azureDisk,omitempty" protobuf:"bytes,22,opt,name=azureDisk"`
	// photonPersistentDisk represents a PhotonController persistent disk attached and mounted on kubelets host machine
	PhotonPersistentDisk *PhotonPersistentDiskVolumeSource `json:"photonPersistentDisk,omitempty" protobuf:"bytes,23,opt,name=photonPersistentDisk"`
	// projected items for all in one resources secrets, configmaps, and downward API
	Projected *ProjectedVolumeSource `json:"projected,omitempty" protobuf:"bytes,26,opt,name=projected"`
	// portworxVolume represents a portworx volume attached and mounted on kubelets host machine
	// +optional
	PortworxVolume *PortworxVolumeSource `json:"portworxVolume,omitempty" protobuf:"bytes,24,opt,name=portworxVolume"`
	// scaleIO represents a ScaleIO persistent volume attached and mounted on Kubernetes nodes.
	// +optional
	ScaleIO *ScaleIOVolumeSource `json:"scaleIO,omitempty" protobuf:"bytes,25,opt,name=scaleIO"`
	// storageOS represents a StorageOS volume attached and mounted on Kubernetes nodes.
	// +optional
	StorageOS *StorageOSVolumeSource `json:"storageos,omitempty" protobuf:"bytes,27,opt,name=storageos"`
	// csi (Container Storage Interface) represents ephemeral storage that is handled by certain external CSI drivers (Beta feature).
	// +optional
	CSI *CSIVolumeSource `json:"csi,omitempty" protobuf:"bytes,28,opt,name=csi"`
	// ephemeral represents a volume that is handled by a cluster storage driver.
	// The volume's lifecycle is tied to the pod that defines it - it will be created before the pod starts,
	// and deleted when the pod is removed.
	//
	// Use this if:
	// a) the volume is only needed while the pod runs,
	// b) features of normal volumes like restoring from snapshot or capacity
	//    tracking are needed,
	// c) the storage driver is specified through a storage class, and
	// d) the storage driver supports dynamic volume provisioning through
	//    a PersistentVolumeClaim (see EphemeralVolumeSource for more
	//    information on the connection between this volume type
	//    and PersistentVolumeClaim).
	//
	// Use PersistentVolumeClaim or one of the vendor-specific
	// APIs for volumes that persist for longer than the lifecycle
	// of an individual pod.
	//
	// Use CSI for light-weight local ephemeral volumes if the CSI driver is meant to
	// be used that way - see the documentation of the driver for
	// more information.
	//
	// A pod can use both types of ephemeral volumes and
	// persistent volumes at the same time.
	//
	// +optional
	Ephemeral *EphemeralVolumeSource `json:"ephemeral,omitempty" protobuf:"bytes,29,opt,name=ephemeral"`
}
```


v1.Volume struct
```go
//在yaml文件中volumes属性指定会传到这里
type Volume struct {
	// name of the volume.
	// Must be a DNS_LABEL and unique within the pod.
	// More info: https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names
	Name string `json:"name" protobuf:"bytes,1,opt,name=name"`
	// volumeSource represents the location and type of the mounted volume.
	// If not specified, the Volume is implied to be an EmptyDir.
	// This implied behavior is deprecated and will be removed in a future version.
	VolumeSource `json:",inline" protobuf:"bytes,2,opt,name=volumeSource"`
}
```

volume.Spec
```go
// Spec is an internal representation of a volume.  All API volume types translate to Spec.
type Spec struct {
	Volume                          *v1.Volume
	PersistentVolume                *v1.PersistentVolume
	ReadOnly                        bool
	InlineVolumeSpecForCSIMigration bool
	Migrated                        bool
}
```

PersistentVolume
```go
//还没搞懂这几个类型
type PersistentVolume struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// spec defines a specification of a persistent volume owned by the cluster.
	// Provisioned by an administrator.
	// More info: https://kubernetes.io/docs/concepts/storage/persistent-volumes#persistent-volumes
	// +optional
	Spec PersistentVolumeSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// status represents the current information/status for the persistent volume.
	// Populated by the system.
	// Read-only.
	// More info: https://kubernetes.io/docs/concepts/storage/persistent-volumes#persistent-volumes
	// +optional
	Status PersistentVolumeStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

metav1.TypeMeta
```go
type TypeMeta struct {
	// Kind is a string value representing the REST resource this object represents.
	// Servers may infer this from the endpoint the client submits requests to.
	// Cannot be updated.
	// In CamelCase.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
	// +optional
	Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`

	// APIVersion defines the versioned schema of this representation of an object.
	// Servers should convert recognized schemas to the latest internal value, and
	// may reject unrecognized values.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
	// +optional
	APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`
}
```

metav1.ObjectMeta
```go
type ObjectMeta struct {
	// Name must be unique within a namespace. Is required when creating resources, although
	// some resources may allow a client to request the generation of an appropriate name
	// automatically. Name is primarily intended for creation idempotence and configuration
	// definition.
	// Cannot be updated.
	// More info: http://kubernetes.io/docs/user-guide/identifiers#names
	// +optional
	Name string `json:"name,omitempty" protobuf:"bytes,1,opt,name=name"`

	// GenerateName is an optional prefix, used by the server, to generate a unique
	// name ONLY IF the Name field has not been provided.
	// If this field is used, the name returned to the client will be different
	// than the name passed. This value will also be combined with a unique suffix.
	// The provided value has the same validation rules as the Name field,
	// and may be truncated by the length of the suffix required to make the value
	// unique on the server.
	//
	// If this field is specified and the generated name exists, the server will return a 409.
	//
	// Applied only if Name is not specified.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#idempotency
	// +optional
	GenerateName string `json:"generateName,omitempty" protobuf:"bytes,2,opt,name=generateName"`

	// Namespace defines the space within which each name must be unique. An empty namespace is
	// equivalent to the "default" namespace, but "default" is the canonical representation.
	// Not all objects are required to be scoped to a namespace - the value of this field for
	// those objects will be empty.
	//
	// Must be a DNS_LABEL.
	// Cannot be updated.
	// More info: http://kubernetes.io/docs/user-guide/namespaces
	// +optional
	Namespace string `json:"namespace,omitempty" protobuf:"bytes,3,opt,name=namespace"`

	// Deprecated: selfLink is a legacy read-only field that is no longer populated by the system.
	// +optional
	SelfLink string `json:"selfLink,omitempty" protobuf:"bytes,4,opt,name=selfLink"`

	// UID is the unique in time and space value for this object. It is typically generated by
	// the server on successful creation of a resource and is not allowed to change on PUT
	// operations.
	//
	// Populated by the system.
	// Read-only.
	// More info: http://kubernetes.io/docs/user-guide/identifiers#uids
	// +optional
	UID types.UID `json:"uid,omitempty" protobuf:"bytes,5,opt,name=uid,casttype=k8s.io/kubernetes/pkg/types.UID"`

	// An opaque value that represents the internal version of this object that can
	// be used by clients to determine when objects have changed. May be used for optimistic
	// concurrency, change detection, and the watch operation on a resource or set of resources.
	// Clients must treat these values as opaque and passed unmodified back to the server.
	// They may only be valid for a particular resource or set of resources.
	//
	// Populated by the system.
	// Read-only.
	// Value must be treated as opaque by clients and .
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#concurrency-control-and-consistency
	// +optional
	ResourceVersion string `json:"resourceVersion,omitempty" protobuf:"bytes,6,opt,name=resourceVersion"`

	// A sequence number representing a specific generation of the desired state.
	// Populated by the system. Read-only.
	// +optional
	Generation int64 `json:"generation,omitempty" protobuf:"varint,7,opt,name=generation"`

	// CreationTimestamp is a timestamp representing the server time when this object was
	// created. It is not guaranteed to be set in happens-before order across separate operations.
	// Clients may not set this value. It is represented in RFC3339 form and is in UTC.
	//
	// Populated by the system.
	// Read-only.
	// Null for lists.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	CreationTimestamp Time `json:"creationTimestamp,omitempty" protobuf:"bytes,8,opt,name=creationTimestamp"`

	// DeletionTimestamp is RFC 3339 date and time at which this resource will be deleted. This
	// field is set by the server when a graceful deletion is requested by the user, and is not
	// directly settable by a client. The resource is expected to be deleted (no longer visible
	// from resource lists, and not reachable by name) after the time in this field, once the
	// finalizers list is empty. As long as the finalizers list contains items, deletion is blocked.
	// Once the deletionTimestamp is set, this value may not be unset or be set further into the
	// future, although it may be shortened or the resource may be deleted prior to this time.
	// For example, a user may request that a pod is deleted in 30 seconds. The Kubelet will react
	// by sending a graceful termination signal to the containers in the pod. After that 30 seconds,
	// the Kubelet will send a hard termination signal (SIGKILL) to the container and after cleanup,
	// remove the pod from the API. In the presence of network partitions, this object may still
	// exist after this timestamp, until an administrator or automated process can determine the
	// resource is fully terminated.
	// If not set, graceful deletion of the object has not been requested.
	//
	// Populated by the system when a graceful deletion is requested.
	// Read-only.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	DeletionTimestamp *Time `json:"deletionTimestamp,omitempty" protobuf:"bytes,9,opt,name=deletionTimestamp"`

	// Number of seconds allowed for this object to gracefully terminate before
	// it will be removed from the system. Only set when deletionTimestamp is also set.
	// May only be shortened.
	// Read-only.
	// +optional
	DeletionGracePeriodSeconds *int64 `json:"deletionGracePeriodSeconds,omitempty" protobuf:"varint,10,opt,name=deletionGracePeriodSeconds"`

	// Map of string keys and values that can be used to organize and categorize
	// (scope and select) objects. May match selectors of replication controllers
	// and services.
	// More info: http://kubernetes.io/docs/user-guide/labels
	// +optional
	Labels map[string]string `json:"labels,omitempty" protobuf:"bytes,11,rep,name=labels"`

	// Annotations is an unstructured key value map stored with a resource that may be
	// set by external tools to store and retrieve arbitrary metadata. They are not
	// queryable and should be preserved when modifying objects.
	// More info: http://kubernetes.io/docs/user-guide/annotations
	// +optional
	Annotations map[string]string `json:"annotations,omitempty" protobuf:"bytes,12,rep,name=annotations"`

	// List of objects depended by this object. If ALL objects in the list have
	// been deleted, this object will be garbage collected. If this object is managed by a controller,
	// then an entry in this list will point to this controller, with the controller field set to true.
	// There cannot be more than one managing controller.
	// +optional
	// +patchMergeKey=uid
	// +patchStrategy=merge
	OwnerReferences []OwnerReference `json:"ownerReferences,omitempty" patchStrategy:"merge" patchMergeKey:"uid" protobuf:"bytes,13,rep,name=ownerReferences"`

	// Must be empty before the object is deleted from the registry. Each entry
	// is an identifier for the responsible component that will remove the entry
	// from the list. If the deletionTimestamp of the object is non-nil, entries
	// in this list can only be removed.
	// Finalizers may be processed and removed in any order.  Order is NOT enforced
	// because it introduces significant risk of stuck finalizers.
	// finalizers is a shared field, any actor with permission can reorder it.
	// If the finalizer list is processed in order, then this can lead to a situation
	// in which the component responsible for the first finalizer in the list is
	// waiting for a signal (field value, external system, or other) produced by a
	// component responsible for a finalizer later in the list, resulting in a deadlock.
	// Without enforced ordering finalizers are free to order amongst themselves and
	// are not vulnerable to ordering changes in the list.
	// +optional
	// +patchStrategy=merge
	Finalizers []string `json:"finalizers,omitempty" patchStrategy:"merge" protobuf:"bytes,14,rep,name=finalizers"`

	// Deprecated: ClusterName is a legacy field that was always cleared by
	// the system and never used; it will be removed completely in 1.25.
	//
	// The name in the go struct is changed to help clients detect
	// accidental use.
	//
	// +optional
	ZZZ_DeprecatedClusterName string `json:"clusterName,omitempty" protobuf:"bytes,15,opt,name=clusterName"`

	// ManagedFields maps workflow-id and version to the set of fields
	// that are managed by that workflow. This is mostly for internal
	// housekeeping, and users typically shouldn't need to set or
	// understand this field. A workflow can be the user's name, a
	// controller's name, or the name of a specific apply path like
	// "ci-cd". The set of fields is always in the version that the
	// workflow used when modifying the object.
	//
	// +optional
	ManagedFields []ManagedFieldsEntry `json:"managedFields,omitempty" protobuf:"bytes,17,rep,name=managedFields"`
}
```

v1.Pod
```go
type Pod struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Specification of the desired behavior of the pod.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
	// +optional
	// 与Pod有关的描述
	Spec PodSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// Most recently observed status of the pod.
	// This data may not be up to date.
	// Populated by the system.
	// Read-only.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
	// +optional
	Status PodStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

VolumeToMount 
```go
type VolumeToMount struct {
	// VolumeName is the unique identifier for the volume that should be
	// mounted.
	// 就是string类型
	VolumeName v1.UniqueVolumeName

	// PodName is the unique identifier for the pod that the volume should be
	// mounted to after it is attached.
	// UID string
	PodName volumetypes.UniquePodName

	// VolumeSpec is a volume spec containing the specification for the volume
	// that should be mounted. Used to create NewMounter. Used to generate
	// InnerVolumeSpecName.

	VolumeSpec *volume.Spec

	// outerVolumeSpecName is the podSpec.Volume[x].Name of the volume. If the
	// volume was referenced through a persistent volume claim, this contains
	// the podSpec.Volume[x].Name of the persistent volume claim.
	OuterVolumeSpecName string

	// Pod to mount the volume to. Used to create NewMounter.
	Pod *v1.Pod

	// PluginIsAttachable indicates that the plugin for this volume implements
	// the volume.Attacher interface
	PluginIsAttachable bool

	// PluginIsDeviceMountable indicates that the plugin for this volume implements
	// the volume.DeviceMounter interface
	PluginIsDeviceMountable bool

	// VolumeGidValue contains the value of the GID annotation, if present.
	VolumeGidValue string

	// DevicePath contains the path on the node where the volume is attached.
	// For non-attachable volumes this is empty.
	DevicePath string

	// ReportedInUse indicates that the volume was successfully added to the
	// VolumesInUse field in the node's status.
	ReportedInUse bool

	// DesiredSizeLimit indicates the desired upper bound on the size of the volume
	// (if so implemented)
	DesiredSizeLimit *resource.Quantity

	// time at which volume was requested to be mounted
	MountRequestTime time.Time

	// PersistentVolumeSize stores desired size of the volume.
	// usually this is the size if pv.Spec.Capacity
	PersistentVolumeSize resource.Quantity
}
```



MountedVolume
```go
// MountedVolume represents a volume that has successfully been mounted to a pod.
type MountedVolume struct {
	// PodName is the unique identifier of the pod mounted to.
	PodName volumetypes.UniquePodName

	// VolumeName is the unique identifier of the volume mounted to the pod.
	VolumeName v1.UniqueVolumeName

	// InnerVolumeSpecName is the volume.Spec.Name() of the volume. If the
	// volume was referenced through a persistent volume claims, this contains
	// the name of the bound persistent volume object.
	// It is the name that plugins use in their pod mount path, i.e.
	// /var/lib/kubelet/pods/{podUID}/volumes/{escapeQualifiedPluginName}/{innerVolumeSpecName}/
	// PVC example,
	//   apiVersion: v1
	//   kind: PersistentVolume
	//   metadata:
	//     name: pv0003				<- InnerVolumeSpecName
	//   spec:
	//     capacity:
	//       storage: 5Gi
	//     accessModes:
	//       - ReadWriteOnce
	//     persistentVolumeReclaimPolicy: Recycle
	//     nfs:
	//       path: /tmp
	//       server: 172.17.0.2
	// Non-PVC example:
	//   apiVersion: v1
	//   kind: Pod
	//   metadata:
	//     name: test-pd
	//   spec:
	//     containers:
	//     - image: registry.k8s.io/test-webserver
	//     	 name: test-container
	//     	 volumeMounts:
	//     	 - mountPath: /test-pd
	//     	   name: test-volume
	//     volumes:
	//     - name: test-volume			<- InnerVolumeSpecName
	//     	 gcePersistentDisk:
	//     	   pdName: my-data-disk
	//     	   fsType: ext4
	InnerVolumeSpecName string

	// outerVolumeSpecName is the podSpec.Volume[x].Name of the volume. If the
	// volume was referenced through a persistent volume claim, this contains
	// the podSpec.Volume[x].Name of the persistent volume claim.
	// PVC example:
	//   kind: Pod
	//   apiVersion: v1
	//   metadata:
	//     name: mypod
	//   spec:
	//     containers:
	//       - name: myfrontend
	//         image: dockerfile/nginx
	//         volumeMounts:
	//         - mountPath: "/var/www/html"
	//           name: mypd
	//     volumes:
	//       - name: mypd				<- OuterVolumeSpecName
	//         persistentVolumeClaim:
	//           claimName: myclaim
	// Non-PVC example:
	//   apiVersion: v1
	//   kind: Pod
	//   metadata:
	//     name: test-pd
	//   spec:
	//     containers:
	//     - image: registry.k8s.io/test-webserver
	//     	 name: test-container
	//     	 volumeMounts:
	//     	 - mountPath: /test-pd
	//     	   name: test-volume
	//     volumes:
	//     - name: test-volume			<- OuterVolumeSpecName
	//     	 gcePersistentDisk:
	//     	   pdName: my-data-disk
	//     	   fsType: ext4
	OuterVolumeSpecName string

	// PluginName is the "Unescaped Qualified" name of the volume plugin used to
	// mount and unmount this volume. It can be used to fetch the volume plugin
	// to unmount with, on demand. It is also the name that plugins use, though
	// escaped, in their pod mount path, i.e.
	// /var/lib/kubelet/pods/{podUID}/volumes/{escapeQualifiedPluginName}/{outerVolumeSpecName}/
	PluginName string

	// PodUID is the UID of the pod mounted to. It is also the string used by
	// plugins in their pod mount path, i.e.
	// /var/lib/kubelet/pods/{podUID}/volumes/{escapeQualifiedPluginName}/{outerVolumeSpecName}/
	PodUID types.UID

	// Mounter is the volume mounter used to mount this volume. It is required
	// by kubelet to create container.VolumeMap.
	// Mounter is only required for file system volumes and not required for block volumes.
	Mounter volume.Mounter

	// BlockVolumeMapper is the volume mapper used to map this volume. It is required
	// by kubelet to create container.VolumeMap.
	// BlockVolumeMapper is only required for block volumes and not required for file system volumes.
	BlockVolumeMapper volume.BlockVolumeMapper

	// VolumeGidValue contains the value of the GID annotation, if present.
	VolumeGidValue string

	// VolumeSpec is a volume spec containing the specification for the volume
	// that should be mounted.
	VolumeSpec *volume.Spec

	// DeviceMountPath contains the path on the node where the device should
	// be mounted after it is attached.
	DeviceMountPath string
}
```

Mounter
```go
type Mounter interface {
	// Uses Interface to provide the path for Docker binds.
	//
	Volume

	// SetUp prepares and mounts/unpacks the volume to a
	// self-determined directory path. The mount point and its
	// content should be owned by `fsUser` or 'fsGroup' so that it can be
	// accessed by the pod. This may be called more than once, so
	// implementations must be idempotent.
	// It could return following types of errors:
	//   - TransientOperationFailure
	//   - UncertainProgressError
	//   - Error of any other type should be considered a final error
	SetUp(mounterArgs MounterArgs) error

	// SetUpAt prepares and mounts/unpacks the volume to the
	// specified directory path, which may or may not exist yet.
	// The mount point and its content should be owned by `fsUser`
	// 'fsGroup' so that it can be accessed by the pod. This may
	// be called more than once, so implementations must be
	// idempotent.
	SetUpAt(dir string, mounterArgs MounterArgs) error
	// GetAttributes returns the attributes of the mounter.
	// This function is called after SetUp()/SetUpAt().
	GetAttributes() Attributes
}
```

hostPathMounter
```go
type hostPathMounter struct {
	*hostPath
	readOnly      bool
	mounter       mount.Interface
	hu            hostutil.HostUtils
	noTypeChecker bool
}
```

```go
type hostPath struct {
	path     string
	pathType *v1.HostPathType
	volume.MetricsNil
}
```


Volume interface
```go
type Volume interface {
	// GetPath returns the path to which the volume should be mounted for the
	// pod.
	GetPath() string

	// MetricsProvider embeds methods for exposing metrics (e.g.
	// used, available space).
	MetricsProvider
}
```

volume.VolumeOptions
VolumeOptions contains option information about a volume.
```go
type VolumeOptions struct {

	// Reclamation policy for a persistent volume
	PersistentVolumeReclaimPolicy v1.PersistentVolumeReclaimPolicy
	// Mount options for a persistent volume
	MountOptions []string
	// Suggested PV.Name of the PersistentVolume to provision.
	// This is a generated name guaranteed to be unique in Kubernetes cluster.
	// If you choose not to use it as volume name, ensure uniqueness by either
	// combining it with your value or create unique values of your own.
	PVName string
	// PVC is reference to the claim that lead to provisioning of a new PV.
	// Provisioners *must* create a PV that would be matched by this PVC,
	// i.e. with required capacity, accessMode, labels matching PVC.Selector and
	// so on.
	PVC *v1.PersistentVolumeClaim
	// Unique name of Kubernetes cluster.
	ClusterName string
	// Tags to attach to the real volume in the cloud provider - e.g. AWS EBS
	CloudTags *map[string]string
	// Volume provisioning parameters from StorageClass
	Parameters map[string]string
}
```

VolumePlugin interface
```go
// VolumePlugin is an interface to volume plugins that can be used on a
// kubernetes node (e.g. by kubelet) to instantiate and manage volumes.
type VolumePlugin interface {
	// Init initializes the plugin.  This will be called exactly once
	// before any New* calls are made - implementations of plugins may
	// depend on this.
	Init(host VolumeHost) error

	// Name returns the plugin's name.  Plugins must use namespaced names
	// such as "example.com/volume" and contain exactly one '/' character.
	// The "kubernetes.io" namespace is reserved for plugins which are
	// bundled with kubernetes.
	GetPluginName() string

	// GetVolumeName returns the name/ID to uniquely identifying the actual
	// backing device, directory, path, etc. referenced by the specified volume
	// spec.
	// For Attachable volumes, this value must be able to be passed back to
	// volume Detach methods to identify the device to act on.
	// If the plugin does not support the given spec, this returns an error.
	GetVolumeName(spec *Spec) (string, error)

	// CanSupport tests whether the plugin supports a given volume
	// specification from the API.  The spec pointer should be considered
	// const.
	CanSupport(spec *Spec) bool

	// RequiresRemount returns true if this plugin requires mount calls to be
	// reexecuted. Atomically updating volumes, like Downward API, depend on
	// this to update the contents of the volume.
	RequiresRemount(spec *Spec) bool

	// NewMounter creates a new volume.Mounter from an API specification.
	// Ownership of the spec pointer in *not* transferred.
	// - spec: The v1.Volume spec
	// - pod: The enclosing pod
	NewMounter(spec *Spec, podRef *v1.Pod, opts VolumeOptions) (Mounter, error)

	// NewUnmounter creates a new volume.Unmounter from recoverable state.
	// - name: The volume name, as per the v1.Volume spec.
	// - podUID: The UID of the enclosing pod
	NewUnmounter(name string, podUID types.UID) (Unmounter, error)

	// ConstructVolumeSpec constructs a volume spec based on the given volume name
	// and volumePath. The spec may have incomplete information due to limited
	// information from input. This function is used by volume manager to reconstruct
	// volume spec by reading the volume directories from disk
	ConstructVolumeSpec(volumeName, volumePath string) (*Spec, error)

	// SupportsMountOption returns true if volume plugins supports Mount options
	// Specifying mount options in a volume plugin that doesn't support
	// user specified mount options will result in error creating persistent volumes
	SupportsMountOption() bool

	// SupportsBulkVolumeVerification checks if volume plugin type is capable
	// of enabling bulk polling of all nodes. This can speed up verification of
	// attached volumes by quite a bit, but underlying pluging must support it.
	SupportsBulkVolumeVerification() bool
}
```

VolumePluginMgr 
```go
// VolumePluginMgr tracks registered plugins.
type VolumePluginMgr struct {
	mutex                     sync.RWMutex
	plugins                   map[string]VolumePlugin
	prober                    DynamicPluginProber
	probedPlugins             map[string]VolumePlugin
	loggedDeprecationWarnings sets.String
	Host                      VolumeHost
}
```

NodeResizeOptions
```go
type NodeResizeOptions struct {
	VolumeSpec *Spec

	// DevicePath - location of actual device on the node. In case of CSI
	// this just could be volumeID
	DevicePath string

	// DeviceMountPath location where device is mounted on the node. If volume type
	// is attachable - this would be global mount path otherwise
	// it would be location where volume was mounted for the pod
	DeviceMountPath string

	// DeviceStagingPath stores location where the volume is staged
	DeviceStagePath string

	NewSize resource.Quantity
	OldSize resource.Quantity
}
```


MounterArgs
```go
type MounterArgs struct {
	// When FsUser is set, the ownership of the volume will be modified to be
	// owned and writable by FsUser. Otherwise, there is no side effects.
	// Currently only supported with projected service account tokens.
	FsUser              *int64
	FsGroup             *int64
	// FSGroupChangePolicy和DesiredSize string
	FSGroupChangePolicy *v1.PodFSGroupChangePolicy
	DesiredSize         *resource.Quantity
}
```


StorageMedium
```go
const (
	StorageMediumDefault         StorageMedium = ""           // use whatever the default is for the node, assume anything we don't explicitly handle is this
	StorageMediumMemory          StorageMedium = "Memory"     // use memory (e.g. tmpfs on linux)
	StorageMediumHugePages       StorageMedium = "HugePages"  // use hugepages
	StorageMediumHugePagesPrefix StorageMedium = "HugePages-" // prefix for full medium notation HugePages-<size>
)
```

nestedPendingOperations
```go
type nestedPendingOperations struct {
	operations                []operation
	exponentialBackOffOnError bool
	cond                      *sync.Cond
	lock                      sync.RWMutex
}
```

operation
```go
type operation struct {
	key              operationKey
	operationName    string
	// 待定的操作
	operationPending bool
	expBackoff       exponentialbackoff.ExponentialBackoff
}
```

operationKey
```go
type operationKey struct {
    volumeName v1.UniqueVolumeName
    podName    volumetypes.UniquePodName
    nodeName   types.NodeName
}
```

volumetypes.GeneratedOperations
```go
type GeneratedOperations struct {
	// Name of operation - could be used for resetting shared exponential backoff
	OperationName     string
	OperationFunc     func() (context OperationContext)
	EventRecorderFunc func(*error)
	CompleteFunc      func(CompleteFuncParam)
}
```


```go
type OperationContext struct {
	EventErr    error
	DetailedErr error
	Migrated    bool
}
```



