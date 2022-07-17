##


```go
// in pkg/kubelet/kubelet_pods.go
// makeMounts determines the mount points for the given container.
func makeMounts(pod *v1.Pod, podDir string, container *v1.Container, hostName, hostDomain, podIP string, podVolumes kubecontainer.VolumeMap) ([]kubecontainer.Mount, error) {
    // ...
	mounts := []kubecontainer.Mount{}
	for _, mount := range container.VolumeMounts {
        // ...
		hostPath, err := volume.GetPath(vol.Mounter)
		if err != nil {
			return nil, err
		}
		if mount.SubPath != "" {
			if filepath.IsAbs(mount.SubPath) {
				return nil, fmt.Errorf("error SubPath `%s` must not be an absolute path", mount.SubPath)
			}

			err = volumevalidation.ValidatePathNoBacksteps(mount.SubPath)
			if err != nil {
				return nil, fmt.Errorf("unable to provision SubPath `%s`: %v", mount.SubPath, err)
			}

			fileinfo, err := os.Lstat(hostPath)
			if err != nil {
				return nil, err
			}
			perm := fileinfo.Mode()
			// 关键点1
      		// 将hostPath和SubPath字符串合并
			hostPath = filepath.Join(hostPath, mount.SubPath)
		// ...
		}
		// 修复之后在这中间增加了对subPath的安全检查
		// ...
        // 关键点2
        // 直接将合并后的hostpath 作为HostPath
		mounts = append(mounts, kubecontainer.Mount{
			Name:           mount.Name,
			ContainerPath:  containerPath,
			HostPath:       hostPath,
			ReadOnly:       mount.ReadOnly,
			SELinuxRelabel: relabelVolume,
			Propagation:    propagation,
		})
	}
    // ...
	return mounts, nil
}
```

makeMounts in kubenetes/pkg/kubelet/kuberruntime/kuberuntime_container.go
```go
// in kubenetes/pkg/kubelet/kuberruntime/kuberuntime_container.go
// makeMounts generates container volume mounts for kubelet runtime v1.
func (m *kubeGenericRuntimeManager) makeMounts(opts *kubecontainer.RunContainerOptions, container *v1.Container) []*runtimeapi.Mount {
	volumeMounts := []*runtimeapi.Mount{}
	//遍历每一个要mount的volume
	for idx := range opts.Mounts {
		v := opts.Mounts[idx]
		//取出这个SelinuxRelabel并检查是否启动selinux
		selinuxRelabel := v.SELinuxRelabel && selinux.GetEnabled()
		//将yaml里面配置的各种属性传进来
		mount := &runtimeapi.Mount{
			HostPath:       v.HostPath,
			ContainerPath:  v.ContainerPath,
			Readonly:       v.ReadOnly,
			SelinuxRelabel: selinuxRelabel,
			Propagation:    v.Propagation,
		}
		//将配置好的mount属性加入volumeMounts
		volumeMounts = append(volumeMounts, mount)
	}

	// The reason we create and mount the log file in here (not in kubelet) is because
	// the file's location depends on the ID of the container, and we need to create and
	// mount the file before actually starting the container.
	// 需要在启动容器之前创建并挂在log file
	// opts.PodContainerDir string 表示若容器有指定TerminationMessagePath则使用它作为log file
	if opts.PodContainerDir != "" && len(container.TerminationMessagePath) != 0 {
		// Because the PodContainerDir contains pod uid and container name which is unique enough,
		// here we just add a random id to make the path unique for different instances
		// of the same container.
		//	cid作为容器的唯一标志
		cid := makeUID()
		containerLogPath := filepath.Join(opts.PodContainerDir, cid)
		fs, err := m.osInterface.Create(containerLogPath)
		if err != nil {
			utilruntime.HandleError(fmt.Errorf("error on creating termination-log file %q: %v", containerLogPath, err))
		} else {
			fs.Close()

			// Chmod is needed because ioutil.WriteFile() ends up calling
			// open(2) to create the file, so the final mode used is "mode &
			// ~umask". But we want to make sure the specified mode is used
			// in the file no matter what the umask is.
			if err := m.osInterface.Chmod(containerLogPath, 0666); err != nil {
				utilruntime.HandleError(fmt.Errorf("unable to set termination-log file permissions %q: %v", containerLogPath, err))
			}

			// Volume Mounts fail on Windows if it is not of the form C:/
			containerLogPath = volumeutil.MakeAbsolutePath(goruntime.GOOS, containerLogPath)
			terminationMessagePath := volumeutil.MakeAbsolutePath(goruntime.GOOS, container.TerminationMessagePath)
			selinuxRelabel := selinux.GetEnabled()
			//把containerLogPath也挂载
			volumeMounts = append(volumeMounts, &runtimeapi.Mount{
				HostPath:       containerLogPath,
				ContainerPath:  terminationMessagePath,
				SelinuxRelabel: selinuxRelabel,
			})
		}
	}
	//最后返回VolumeMount的数组
	return volumeMounts
}
```


makeMounts in kubernetes/pkg/kubelet_pods.go
```go
// in kubernetes/pkg/kubelet_pods.go
// makeMounts determines the mount points for the given container.
// makeMount的作用就是把所有要Mount的Volume处理好
func makeMounts(pod *v1.Pod, podDir string, container *v1.Container, hostName, hostDomain string, podIPs []string, podVolumes kubecontainer.VolumeMap, hu hostutil.HostUtils, subpather subpath.Interface, expandEnvs []kubecontainer.EnvVar) ([]kubecontainer.Mount, func(), error) {
	//
	mountEtcHostsFile := shouldMountHostsFile(pod, podIPs)
	klog.V(3).InfoS("Creating hosts mount for container", "pod", klog.KObj(pod), "containerName", container.Name, "podIPs", podIPs, "path", mountEtcHostsFile)
	//	Name ContainerPath HostPath string类型
	//	ReadOnly SELinuxRelabel bool类型
	//	Propagation mount的传播方式 0 1 2三种
	//	私人 双向 host to container
	//	https://pkg.go.dev/k8s.io/cri-api@v0.24.2/pkg/apis/runtime/v1#MountPropagation
	mounts := []kubecontainer.Mount{}
	var cleanupAction func()
	//	container.VolumeMounts []VolumeMount类型
	//	Name MountPath SubPath SubPathExpr string类型
	//	ReadOnly bool类型
	//	MountPropagation
	//	mount就是VolumeMount
	for i, mount := range container.VolumeMounts {
		// do not mount /etc/hosts if container is already mounting on the path
		//	
		mountEtcHostsFile = mountEtcHostsFile && (mount.MountPath != etcHostsPath)
		//	podVolumes是Kubecontainer.VolumeMap类型
		//	type VolumeMap map[string]VolumeInfo
		//	podVolumes[mount.Name]表示将整个名字的VolumeInfo取出来给vol
		vol, ok := podVolumes[mount.Name]
		if !ok || vol.Mounter == nil {
			klog.ErrorS(nil, "Mount cannot be satisfied for the container, because the volume is missing or the volume mounter (vol.Mounter) is nil",
				"containerName", container.Name, "ok", ok, "volumeMounter", mount)
			return nil, cleanupAction, fmt.Errorf("cannot find volume %q to mount into container %q", mount.Name, container.Name)
		}

		relabelVolume := false
		//	If the volume supports SELinux and it has not been
		//	relabeled already and it is not a read-only volume,
		//	relabel it and mark it as labeled
		//	支持SELinux、没有被relabel而且还不是只读的volume就relabel它
		//	vol是VolumeInfo类型
		//	Mounter是volume.Mounter
		//	根据volume类型的不同会调用不同的GetAttributes
		//	Attributes有三个属性:ReadOnly Managed SELinuxRelabel bool
		
		if vol.Mounter.GetAttributes().Managed && vol.Mounter.GetAttributes().SELinuxRelabel && !vol.SELinuxLabeled {
			vol.SELinuxLabeled = true
			relabelVolume = true
		}
		
		//GetPath就是检查hostPath是否为空

		hostPath, err := volumeutil.GetPath(vol.Mounter)
		if err != nil {
			return nil, cleanupAction, err
		}
		//	SubPathExpr与SubPath是互斥的,若使用SubPathExpr则直接将它添加进mountPath
		subPath := mount.SubPath
		if mount.SubPathExpr != "" {
			subPath, err = kubecontainer.ExpandContainerVolumeMounts(mount, expandEnvs)

			if err != nil {
				return nil, cleanupAction, err
			}
		}
		//	对subPath处理
		if subPath != "" {
			//	判断是否为绝对路径
			if filepath.IsAbs(subPath) {
				return nil, cleanupAction, fmt.Errorf("error SubPath `%s` must not be an absolute path", subPath)
			}
			//	验证路径中不存在..
			err = volumevalidation.ValidatePathNoBacksteps(subPath)
			if err != nil {
				return nil, cleanupAction, fmt.Errorf("unable to provision SubPath `%s`: %v", subPath, err)
			}
			//	将hostPath和subPath合起来
			volumePath := hostPath
			hostPath = filepath.Join(volumePath, subPath)
			
			//	检验path是否已经存在
			if subPathExists, err := hu.PathExists(hostPath); err != nil {
				klog.ErrorS(nil, "Could not determine if subPath exists, will not attempt to change its permissions", "path", hostPath)
			} else if !subPathExists {
				//	设置subPath Dir相应的权限
				// Create the sub path now because if it's auto-created later when referenced, it may have an
				// incorrect ownership and mode. For example, the sub path directory must have at least g+rwx
				// when the pod specifies an fsGroup, and if the directory is not created here, Docker will
				// later auto-create it with the incorrect mode 0750
				// Make extra care not to escape the volume!
				perm, err := hu.GetMode(volumePath)
				if err != nil {
					return nil, cleanupAction, err
				}
				//	保证创建的subPath Dir 不是基于symlink
				if err := subpather.SafeMakeDir(subPath, volumePath, perm); err != nil {
					// Don't pass detailed error back to the user because it could give information about host filesystem
					klog.ErrorS(err, "Failed to create subPath directory for volumeMount of the container", "containerName", container.Name, "volumeMountName", mount.Name)
					return nil, cleanupAction, fmt.Errorf("failed to create subPath directory for volumeMount %q of container %q", mount.Name, container.Name)
				}
			}
			//	为subPath做准备保证在Volume内部避免逃逸
            //  并且将subPath bind-mount防止race condition
			hostPath, cleanupAction, err = subpather.PrepareSafeSubpath(subpath.Subpath{
				VolumeMountIndex: i,
				Path:             hostPath,
				VolumeName:       vol.InnerVolumeSpecName,
				VolumePath:       volumePath,
				PodDir:           podDir,
				ContainerName:    container.Name,
			})
			if err != nil {
				// Don't pass detailed error back to the user because it could give information about host filesystem
				klog.ErrorS(err, "Failed to prepare subPath for volumeMount of the container", "containerName", container.Name, "volumeMountName", mount.Name)
				return nil, cleanupAction, fmt.Errorf("failed to prepare subPath for volumeMount %q of container %q", mount.Name, container.Name)
			}
		}

		// Docker Volume Mounts fail on Windows if it is not of the form C:/
		if volumeutil.IsWindowsLocalPath(runtime.GOOS, hostPath) {
			hostPath = volumeutil.MakeAbsolutePath(runtime.GOOS, hostPath)
		}

		containerPath := mount.MountPath
		// IsAbs returns false for UNC path/SMB shares/named pipes in Windows. So check for those specifically and skip MakeAbsolutePath
		if !volumeutil.IsWindowsUNCPath(runtime.GOOS, containerPath) && !filepath.IsAbs(containerPath) {
			//	转换为绝对路径
			containerPath = volumeutil.MakeAbsolutePath(runtime.GOOS, containerPath)
		}

		propagation, err := translateMountPropagation(mount.MountPropagation)
		if err != nil {
			return nil, cleanupAction, err
		}
		klog.V(5).InfoS("Mount has propagation", "pod", klog.KObj(pod), "containerName", container.Name, "volumeMountName", mount.Name, "propagation", propagation)
		//	获取ReadOnly Flag
		mustMountRO := vol.Mounter.GetAttributes().ReadOnly

		mounts = append(mounts, kubecontainer.Mount{
			Name:           mount.Name,
			ContainerPath:  containerPath,
			HostPath:       hostPath,
			ReadOnly:       mount.ReadOnly || mustMountRO,
			SELinuxRelabel: relabelVolume,
			Propagation:    propagation,
		})
	}
	if mountEtcHostsFile {
		hostAliases := pod.Spec.HostAliases
		hostsMount, err := makeHostsMount(podDir, podIPs, hostName, hostDomain, hostAliases, pod.Spec.HostNetwork)
		if err != nil {
			return nil, cleanupAction, err
		}
		mounts = append(mounts, *hostsMount)
	}
	return mounts, cleanupAction, nil
}
```

PrepareSafeSubpath
```go
func (sp *subpath) PrepareSafeSubpath(subPath Subpath) (newHostPath string, cleanupAction func(), err error) {
	newHostPath, err = doBindSubPath(sp.mounter, subPath)

	// There is no action when the container starts. Bind-mount will be cleaned
	// when container stops by CleanSubPaths.
	cleanupAction = nil
	return newHostPath, cleanupAction, err
}
```

doBindSubPath
```go
func doBindSubPath(mounter mount.Interface, subpath Subpath) (hostPath string, err error) {
	// Linux, kubelet runs on the host:
	// - safely open the subpath
	// - bind-mount /proc/<pid of kubelet>/fd/<fd> to subpath target
	// User can't change /proc/<pid of kubelet>/fd/<fd> to point to a bad place.

	// Evaluate all symlinks here once for all subsequent functions.
	newVolumePath, err := filepath.EvalSymlinks(subpath.VolumePath)
	if err != nil {
		return "", fmt.Errorf("error resolving symlinks in %q: %v", subpath.VolumePath, err)
	}
	newPath, err := filepath.EvalSymlinks(subpath.Path)
	if err != nil {
		return "", fmt.Errorf("error resolving symlinks in %q: %v", subpath.Path, err)
	}
	klog.V(5).Infof("doBindSubPath %q (%q) for volumepath %q", subpath.Path, newPath, subpath.VolumePath)
	subpath.VolumePath = newVolumePath
	subpath.Path = newPath
    // 打开subpath并返回fd 不允许使用symlink(使用openat()逐个/打开,dirfd参数是打开的文件夹 pathname是要打开的文件) 并且路径必须在目录内
	// openat 当pathname是绝对路径的时候与open一样
	// 当pathname是相对路径时openat以dirfd作为pathname的开始地址
	fd, err := safeOpenSubPath(mounter, subpath)
	if err != nil {
		return "", err
	}
	defer syscall.Close(fd)
    // 给subpath创建一个bind mount 防止被替换
	alreadyMounted, bindPathTarget, err := prepareSubpathTarget(mounter, subpath)
	if err != nil {
		return "", err
	}
	if alreadyMounted {
		return bindPathTarget, nil
	}

	success := false
	defer func() {
		// Cleanup subpath on error
		if !success {
			klog.V(4).Infof("doBindSubPath() failed for %q, cleaning up subpath", bindPathTarget)
			if cleanErr := cleanSubPath(mounter, subpath); cleanErr != nil {
				klog.Errorf("Failed to clean subpath %q: %v", bindPathTarget, cleanErr)
			}
		}
	}()

	kubeletPid := os.Getpid()
	mountSource := fmt.Sprintf("/proc/%d/fd/%v", kubeletPid, fd)

	// Do the bind mount
	options := []string{"bind"}
	mountFlags := []string{"--no-canonicalize"}
	klog.V(5).Infof("bind mounting %q at %q", mountSource, bindPathTarget)
	// 加上--no-canonicalize防止mount对路径进行解析时候被替换，前面已经保证subpath是绝对路径了
	if err = mounter.MountSensitiveWithoutSystemdWithMountFlags(mountSource, bindPathTarget, "" /*fstype*/, options, nil /* sensitiveOptions */, mountFlags); err != nil {
		return "", fmt.Errorf("error mounting %s: %s", subpath.Path, err)
	}
	success = true

	klog.V(3).Infof("Bound SubPath %s into %s", subpath.Path, bindPathTarget)
	return bindPathTarget, nil
}
```

safeOpenSubPath
```go
// This implementation is shared between Linux and NsEnter
func safeOpenSubPath(mounter mount.Interface, subpath Subpath) (int, error) {
	if !mount.PathWithinBase(subpath.Path, subpath.VolumePath) {
		return -1, fmt.Errorf("subpath %q not within volume path %q", subpath.Path, subpath.VolumePath)
	}
	fd, err := doSafeOpen(subpath.Path, subpath.VolumePath)
	if err != nil {
		return -1, fmt.Errorf("error opening subpath %v: %v", subpath.Path, err)
	}
	return fd, nil
}
```

doSafeOpen
```go
// This implementation is shared between Linux and NsEnterMounter
// Open path and return its fd.
// Symlinks are disallowed (pathname must already resolve symlinks),
// and the path must be within the base directory.
func doSafeOpen(pathname string, base string) (int, error) {
	pathname = filepath.Clean(pathname)
	base = filepath.Clean(base)

	// Calculate segments to follow
	// 返回绝对路径
	subpath, err := filepath.Rel(base, pathname)
	if err != nil {
		return -1, err
	}
	// 每个/进行检查
	segments := strings.Split(subpath, string(filepath.Separator))

	// Assumption: base is the only directory that we have under control.
	// Base dir is not allowed to be a symlink.
	parentFD, err := syscall.Open(base, nofollowFlags|unix.O_CLOEXEC, 0)
	if err != nil {
		return -1, fmt.Errorf("cannot open directory %s: %s", base, err)
	}
	defer func() {
		if parentFD != -1 {
			if err = syscall.Close(parentFD); err != nil {
				klog.V(4).Infof("Closing FD %v failed for safeopen(%v): %v", parentFD, pathname, err)
			}
		}
	}()

	childFD := -1
	defer func() {
		if childFD != -1 {
			if err = syscall.Close(childFD); err != nil {
				klog.V(4).Infof("Closing FD %v failed for safeopen(%v): %v", childFD, pathname, err)
			}
		}
	}()

	currentPath := base

	// Follow the segments one by one using openat() to make
	// sure the user cannot change already existing directories into symlinks.
	for _, seg := range segments {
		currentPath = filepath.Join(currentPath, seg)
		if !mount.PathWithinBase(currentPath, base) {
			return -1, fmt.Errorf("path %s is outside of allowed base %s", currentPath, base)
		}

		klog.V(5).Infof("Opening path %s", currentPath)
		childFD, err = syscall.Openat(parentFD, seg, openFDFlags|unix.O_CLOEXEC, 0)
		if err != nil {
			return -1, fmt.Errorf("cannot open %s: %s", currentPath, err)
		}

		var deviceStat unix.Stat_t
		err := unix.Fstat(childFD, &deviceStat)
		if err != nil {
			return -1, fmt.Errorf("error running fstat on %s with %v", currentPath, err)
		}
		fileFmt := deviceStat.Mode & syscall.S_IFMT
		if fileFmt == syscall.S_IFLNK {
			return -1, fmt.Errorf("unexpected symlink found %s", currentPath)
		}

		// Close parentFD
		if err = syscall.Close(parentFD); err != nil {
			return -1, fmt.Errorf("closing fd for %q failed: %v", filepath.Dir(currentPath), err)
		}
		// Set child to new parent
		parentFD = childFD
		childFD = -1
	}

	// We made it to the end, return this fd, don't close it
	finalFD := parentFD
	parentFD = -1

	return finalFD, nil
}

```

prepareSubpathTarget
```go
// prepareSubpathTarget creates target for bind-mount of subpath. It returns
// "true" when the target already exists and something is mounted there.
// Given Subpath must have all paths with already resolved symlinks and with
// paths relevant to kubelet (when it runs in a container).
// This function is called also by NsEnterMounter. It works because
// /var/lib/kubelet is mounted from the host into the container with Kubelet as
// /var/lib/kubelet too.
func prepareSubpathTarget(mounter mount.Interface, subpath Subpath) (bool, string, error) {
	// Early check for already bind-mounted subpath.
	bindPathTarget := getSubpathBindTarget(subpath)
	notMount, err := mount.IsNotMountPoint(mounter, bindPathTarget)
	if err != nil {
		if !os.IsNotExist(err) {
			return false, "", fmt.Errorf("error checking path %s for mount: %s", bindPathTarget, err)
		}
		// Ignore ErrorNotExist: the file/directory will be created below if it does not exist yet.
		notMount = true
	}
	if !notMount {
		// It's already mounted, so check if it's bind-mounted to the same path
		samePath, err := checkSubPathFileEqual(subpath, bindPathTarget)
		if err != nil {
			return false, "", fmt.Errorf("error checking subpath mount info for %s: %s", bindPathTarget, err)
		}
		if !samePath {
			// It's already mounted but not what we want, unmount it
			if err = mounter.Unmount(bindPathTarget); err != nil {
				return false, "", fmt.Errorf("error ummounting %s: %s", bindPathTarget, err)
			}
		} else {
			// It's already mounted
			klog.V(5).Infof("Skipping bind-mounting subpath %s: already mounted", bindPathTarget)
			return true, bindPathTarget, nil
		}
	}

	// bindPathTarget is in /var/lib/kubelet and thus reachable without any
	// translation even to containerized kubelet.
	bindParent := filepath.Dir(bindPathTarget)
	err = os.MkdirAll(bindParent, 0750)
	if err != nil && !os.IsExist(err) {
		return false, "", fmt.Errorf("error creating directory %s: %s", bindParent, err)
	}

	t, err := os.Lstat(subpath.Path)
	if err != nil {
		return false, "", fmt.Errorf("lstat %s failed: %s", subpath.Path, err)
	}

	if t.Mode()&os.ModeDir > 0 {
		if err = os.Mkdir(bindPathTarget, 0750); err != nil && !os.IsExist(err) {
			return false, "", fmt.Errorf("error creating directory %s: %s", bindPathTarget, err)
		}
	} else {
		// bindPathTarget is in /var/lib/kubelet
		// "/bin/touch <bindPathTarget>".
		// A file is enough for all possible targets (symlink, device, pipe,
		// socket, ...), bind-mounting them into a file correctly changes type
		// of the target file.
		if err = ioutil.WriteFile(bindPathTarget, []byte{}, 0640); err != nil {
			return false, "", fmt.Errorf("error creating file %s: %s", bindPathTarget, err)
		}
	}
	return false, bindPathTarget, nil
}

```

Mount MountSensitive
```go
// Mount mounts source to target as fstype with given options. 'source' and 'fstype' must
// be an empty string in case it's not required, e.g. for remount, or for auto filesystem
// type, where kernel handles fstype for you. The mount 'options' is a list of options,
// currently come from mount(8), e.g. "ro", "remount", "bind", etc. If no more option is
// required, call Mount with an empty string list or nil.
func (mounter *Mounter) Mount(source string, target string, fstype string, options []string) error {
	return mounter.MountSensitive(source, target, fstype, options, nil)
}

// MountSensitive is the same as Mount() but this method allows
// sensitiveOptions to be passed in a separate parameter from the normal
// mount options and ensures the sensitiveOptions are never logged. This
// method should be used by callers that pass sensitive material (like
// passwords) as mount options.
func (mounter *Mounter) MountSensitive(source string, target string, fstype string, options []string, sensitiveOptions []string) error {
	// Path to mounter binary if containerized mounter is needed. Otherwise, it is set to empty.
	// All Linux distros are expected to be shipped with a mount utility that a support bind mounts.
	mounterPath := ""
	bind, bindOpts, bindRemountOpts, bindRemountOptsSensitive := MakeBindOptsSensitive(options, sensitiveOptions)
	if bind {
		err := mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, bindOpts, bindRemountOptsSensitive, nil /* mountFlags */, true)
		if err != nil {
			return err
		}
		return mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, bindRemountOpts, bindRemountOptsSensitive, nil /* mountFlags */, true)
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
	return mounter.doMount(mounterPath, defaultMountCommand, source, target, fstype, options, sensitiveOptions, nil /* mountFlags */, true)
}
```

MountSensitiveWithoutSystemd
```go
// in kubernetes/staging/src/k8s.io/mount-utils/mount_linux.go
// MountSensitiveWithoutSystemd is the same as MountSensitive() but disable using systemd mount.
func (mounter *Mounter) MountSensitiveWithoutSystemd(source string, target string, fstype string, options []string, sensitiveOptions []string) error {
	return mounter.MountSensitiveWithoutSystemdWithMountFlags(source, target, fstype, options, sensitiveOptions, nil /* mountFlags */)
}
```

MakeBindOpts MakeBindOptsSensitive
```go

// in kubernetes/staging/src/mount-utils/mount.go
// MakeBindOpts detects whether a bind mount is being requested and makes the remount options to
// use in case of bind mount, due to the fact that bind mount doesn't respect mount options.
// The list equals:
//   options - 'bind' + 'remount' (no duplicate)
func MakeBindOpts(options []string) (bool, []string, []string) {
	bind, bindOpts, bindRemountOpts, _ := MakeBindOptsSensitive(options, nil /* sensitiveOptions */)
	return bind, bindOpts, bindRemountOpts
}

// MakeBindOptsSensitive is the same as MakeBindOpts but this method allows
// sensitiveOptions to be passed in a separate parameter from the normal mount
// options and ensures the sensitiveOptions are never logged. This method should
// be used by callers that pass sensitive material (like passwords) as mount
// options.
func MakeBindOptsSensitive(options []string, sensitiveOptions []string) (bool, []string, []string, []string) {
	// Because we have an FD opened on the subpath bind mount, the "bind" option
	// needs to be included, otherwise the mount target will error as busy if you
	// remount as readonly.
	//
	// As a consequence, all read only bind mounts will no longer change the underlying
	// volume mount to be read only.
	bindRemountOpts := []string{"bind", "remount"}
	bindRemountSensitiveOpts := []string{}
	bind := false
	bindOpts := []string{"bind"}

	// _netdev is a userspace mount option and does not automatically get added when
	// bind mount is created and hence we must carry it over.
	if checkForNetDev(options, sensitiveOptions) {
		bindOpts = append(bindOpts, "_netdev")
	}

	for _, option := range options {
		switch option {
		case "bind":
			bind = true
		case "remount":
		default:
			bindRemountOpts = append(bindRemountOpts, option)
		}
	}

	for _, sensitiveOption := range sensitiveOptions {
		switch sensitiveOption {
		case "bind":
			bind = true
		case "remount":
		default:
			bindRemountSensitiveOpts = append(bindRemountSensitiveOpts, sensitiveOption)
		}
	}

	return bind, bindOpts, bindRemountOpts, bindRemountSensitiveOpts
}
```

MountSensitiveWithoutSystemdWithMountFlags
```go

// in kubernetes/staging/src/k8s.io/mount-utils/mount_linux.go
// MountSensitiveWithoutSystemdWithMountFlags is the same as MountSensitiveWithoutSystemd with additional mount flags.
func (mounter *Mounter) MountSensitiveWithoutSystemdWithMountFlags(source string, target string, fstype string, options []string, sensitiveOptions []string, mountFlags []string) error {
	mounterPath := ""
    // 
	bind, bindOpts, bindRemountOpts, bindRemountOptsSensitive := MakeBindOptsSensitive(options, sensitiveOptions)
    // 如果是bind mount则进入
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
```

MakeMountArgsSensitive MakeMountArgsSensitiveWithMountFlags
```go
// MakeMountArgsSensitive makes the arguments to the mount(8) command.
// sensitiveOptions is an extension of options except they will not be logged (because they may contain sensitive material)
func MakeMountArgsSensitive(source, target, fstype string, options []string, sensitiveOptions []string) (mountArgs []string, mountArgsLogStr string) {
	return MakeMountArgsSensitiveWithMountFlags(source, target, fstype, options, sensitiveOptions, nil /* mountFlags */)
}

// MakeMountArgsSensitiveWithMountFlags makes the arguments to the mount(8) command.
// sensitiveOptions is an extension of options except they will not be logged (because they may contain sensitive material)
// mountFlags are additional mount flags that are not related with the fstype
// and mount options
func MakeMountArgsSensitiveWithMountFlags(source, target, fstype string, options []string, sensitiveOptions []string, mountFlags []string) (mountArgs []string, mountArgsLogStr string) {
	// Build mount command as follows:
	//  mount [$mountFlags] [-t $fstype] [-o $options] [$source] $target
	mountArgs = []string{}
	mountArgsLogStr = ""
	
	// 往mountArgs添加mountFlags 
	mountArgs = append(mountArgs, mountFlags...)
	mountArgsLogStr += strings.Join(mountFlags, " ")
	
	// 往mountArgs添加 -t fstype
	if len(fstype) > 0 {
		mountArgs = append(mountArgs, "-t", fstype)
		mountArgsLogStr += strings.Join(mountArgs, " ")
	}
	if len(options) > 0 || len(sensitiveOptions) > 0 {
		combinedOptions := []string{}
		// 将options 和 sesitiveOptions合成combinedOptions
		combinedOptions = append(combinedOptions, options...)
		combinedOptions = append(combinedOptions, sensitiveOptions...)
		// 往mountArgs添加 -o combinedOptions用, 分隔
		mountArgs = append(mountArgs, "-o", strings.Join(combinedOptions, ","))
		// 删除日志中的sensitiveOptions
		// exclude sensitiveOptions from log string
		mountArgsLogStr += " -o " + sanitizedOptionsForLogging(options, sensitiveOptions)
	}

	// 往mountArgs里添加source
	if len(source) > 0 {
		mountArgs = append(mountArgs, source)
		mountArgsLogStr += " " + source
	}
	// 往mountArgs里添加source
	mountArgs = append(mountArgs, target)
	mountArgsLogStr += " " + target

	return mountArgs, mountArgsLogStr
}
```

doMount
```go
// doMount runs the mount command. mounterPath is the path to mounter binary if containerized mounter is used.
// sensitiveOptions is an extension of options except they will not be logged (because they may contain sensitive material)
// systemdMountRequired is an extension of option to decide whether uses systemd mount.
func (mounter *Mounter) doMount(mounterPath string, mountCmd string, source string, target string, fstype string, options []string, sensitiveOptions []string, mountFlags []string, systemdMountRequired bool) error {
	mountArgs, mountArgsLogStr := MakeMountArgsSensitiveWithMountFlags(source, target, fstype, options, sensitiveOptions, mountFlags)
	if len(mounterPath) > 0 {
		mountArgs = append([]string{mountCmd}, mountArgs...)
		mountArgsLogStr = mountCmd + " " + mountArgsLogStr
		mountCmd = mounterPath
	}

    // withoutststemd 和普通两种
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
```

os/exec
Cmd struct
```go
// Cmd represents an external command being prepared or run.
//
// A Cmd cannot be reused after calling its Run, Output or CombinedOutput
// methods.
type Cmd struct {
	// Path is the path of the command to run.
	//
	// This is the only field that must be set to a non-zero
	// value. If Path is relative, it is evaluated relative
	// to Dir.
	Path string

	// Args holds command line arguments, including the command as Args[0].
	// If the Args field is empty or nil, Run uses {Path}.
	//
	// In typical use, both Path and Args are set by calling Command.
	Args []string

	// Env specifies the environment of the process.
	// Each entry is of the form "key=value".
	// If Env is nil, the new process uses the current process's
	// environment.
	// If Env contains duplicate environment keys, only the last
	// value in the slice for each duplicate key is used.
	// As a special case on Windows, SYSTEMROOT is always added if
	// missing and not explicitly set to the empty string.
	Env []string

	// Dir specifies the working directory of the command.
	// If Dir is the empty string, Run runs the command in the
	// calling process's current directory.
	Dir string

	// Stdin specifies the process's standard input.
	//
	// If Stdin is nil, the process reads from the null device (os.DevNull).
	//
	// If Stdin is an *os.File, the process's standard input is connected
	// directly to that file.
	//
	// Otherwise, during the execution of the command a separate
	// goroutine reads from Stdin and delivers that data to the command
	// over a pipe. In this case, Wait does not complete until the goroutine
	// stops copying, either because it has reached the end of Stdin
	// (EOF or a read error) or because writing to the pipe returned an error.
	Stdin io.Reader

	// Stdout and Stderr specify the process's standard output and error.
	//
	// If either is nil, Run connects the corresponding file descriptor
	// to the null device (os.DevNull).
	//
	// If either is an *os.File, the corresponding output from the process
	// is connected directly to that file.
	//
	// Otherwise, during the execution of the command a separate goroutine
	// reads from the process over a pipe and delivers that data to the
	// corresponding Writer. In this case, Wait does not complete until the
	// goroutine reaches EOF or encounters an error.
	//
	// If Stdout and Stderr are the same writer, and have a type that can
	// be compared with ==, at most one goroutine at a time will call Write.
	Stdout io.Writer
	Stderr io.Writer

	// ExtraFiles specifies additional open files to be inherited by the
	// new process. It does not include standard input, standard output, or
	// standard error. If non-nil, entry i becomes file descriptor 3+i.
	//
	// ExtraFiles is not supported on Windows.
	ExtraFiles []*os.File

	// SysProcAttr holds optional, operating system-specific attributes.
	// Run passes it to os.StartProcess as the os.ProcAttr's Sys field.
	SysProcAttr *syscall.SysProcAttr

	// Process is the underlying process, once started.
	Process *os.Process

	// ProcessState contains information about an exited process,
	// available after a call to Wait or Run.
	ProcessState *os.ProcessState

	ctx             context.Context // nil means none
	lookPathErr     error           // LookPath error, if any.
	finished        bool            // when Wait was called
	childFiles      []*os.File
	closeAfterStart []io.Closer
	closeAfterWait  []io.Closer
	goroutine       []func() error
	errch           chan error // one send per goroutine
	waitDone        chan struct{}
}
```
os/exec
exec.Command
```go
// 生成命令
// Command returns the Cmd struct to execute the named program with
// mountArgs从这边传给系统执行的mount
func Command(name string, arg ...string) *Cmd {
	cmd := &Cmd{
		Path: name,
		Args: append([]string{name}, arg...),
	}
	if filepath.Base(name) == name {
		if lp, err := LookPath(name); err != nil {
			cmd.lookPathErr = err
		} else {
			cmd.Path = lp
		}
	}
	return cmd
}
```

os/exec
cmd.CombinedOutput 执行mount
```go
// CombinedOutput runs the command and returns its combined standard
// output and standard error.
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











































