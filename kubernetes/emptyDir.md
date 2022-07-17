# emptyDir
表示Pod的空目录
## emptyDir.medium(string)
作用为选择存储介质类型
默认为"" 表示使用节点的默认介质
```go
const (
	StorageMediumDefault         StorageMedium = ""           // use whatever the default is for the node, assume anything we don't explicitly handle is this
	// 使用内存
    StorageMediumMemory          StorageMedium = "Memory"     // use memory (e.g. tmpfs on linux)
	// HugePages不太懂什么意思
    StorageMediumHugePages       StorageMedium = "HugePages"  // use hugepages
	StorageMediumHugePagesPrefix StorageMedium = "HugePages-" // prefix for full medium notation HugePages-<size>
)
```

参数在mount调用链中的位置
```go

// kubernetes/pkg/volume/emptydir/empty_dir.go
// SetUp creates new directory.
func (ed *emptyDir) SetUp(mounterArgs volume.MounterArgs) error {
	return ed.SetUpAt(ed.GetPath(), mounterArgs)
}

// SetUpAt creates new directory.
func (ed *emptyDir) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
	// 判断dir是否为mountPoint
	// 其实就是判断dir 和 /dir是不是属于同一个设备
	notMnt, err := ed.mounter.IsLikelyNotMountPoint(dir)
	// Getting an os.IsNotExist err from is a contingency; the directory
	// may not exist yet, in which case, setup should run.
	if err != nil && !os.IsNotExist(err) {
		return err
	}

	// If the plugin readiness file is present for this volume, and the
	// storage medium is the default, then the volume is ready.  If the
	// medium is memory, and a mountpoint is present, then the volume is
	// ready.
	// 把PodPluginDir和voldir拼起来
	readyDir := ed.getMetaDir()
	// IsReady检查给定目录中是否存在一个名为'ready'的常规文件，如果该文件存在，则返回true。
	if volumeutil.IsReady(readyDir) {
		if ed.medium == v1.StorageMediumMemory && !notMnt {
			return nil
		} else if ed.medium == v1.StorageMediumDefault {
			// Further check dir exists
			if _, err := os.Stat(dir); err == nil {
				klog.V(6).InfoS("Dir exists, so check and assign quota if the underlying medium supports quotas", "dir", dir)
				err = ed.assignQuota(dir, mounterArgs.DesiredSize)
				return err
			}
			// This situation should not happen unless user manually delete volume dir.
			// In this case, delete ready file and print a warning for it.
			klog.Warningf("volume ready file dir %s exist, but volume dir %s does not. Remove ready dir", readyDir, dir)
			if err := os.RemoveAll(readyDir); err != nil && !os.IsNotExist(err) {
				klog.Warningf("failed to remove ready dir [%s]: %v", readyDir, err)
			}
		}
	}

	// 根据StorageMedium选择
	switch {
	case ed.medium == v1.StorageMediumDefault:
		err = ed.setupDir(dir)
	case ed.medium == v1.StorageMediumMemory:
		err = ed.setupTmpfs(dir)
	case v1helper.IsHugePageMedium(ed.medium):
		err = ed.setupHugepages(dir)
	default:
		err = fmt.Errorf("unknown storage medium %q", ed.medium)
	}

	// SetVolumeOwnership将指定的卷修改为fsGroup所有，并设置SetGid，
	// 使新创建的文件归fsGroup所有。如果fsGroup为nil，则不执行任何操作。
	volume.SetVolumeOwnership(ed, mounterArgs.FsGroup, nil /*fsGroupChangePolicy*/, volumeutil.FSGroupCompleteHook(ed.plugin, nil))

	// If setting up the quota fails, just log a message but don't actually error out.
	// We'll use the old du mechanism in this case, at least until we support
	// enforcement.
	if err == nil {
		volumeutil.SetReady(ed.getMetaDir())
		err = ed.assignQuota(dir, mounterArgs.DesiredSize)
	}
	return err
}

```


```go
// setupTmpfs creates a tmpfs mount at the specified directory.
func (ed *emptyDir) setupTmpfs(dir string) error {
	if ed.mounter == nil {
		return fmt.Errorf("memory storage requested, but mounter is nil")
	}
	if err := ed.setupDir(dir); err != nil {
		return err
	}
	// Make SetUp idempotent.
	medium, isMnt, _, err := ed.mountDetector.GetMountMedium(dir, ed.medium)
	if err != nil {
		return err
	}
	// If the directory is a mountpoint with medium memory, there is no
	// work to do since we are already in the desired state.
	if isMnt && medium == v1.StorageMediumMemory {
		return nil
	}

	var options []string
    // 默认50%
	// Linux system default is 50% of capacity.
	if ed.sizeLimit != nil && ed.sizeLimit.Value() > 0 {
		options = []string{fmt.Sprintf("size=%d", ed.sizeLimit.Value())}
	}

	klog.V(3).Infof("pod %v: mounting tmpfs for volume %v", ed.pod.UID, ed.volName)
	// 执行mount
	return ed.mounter.MountSensitiveWithoutSystemd("tmpfs", dir, "tmpfs", options, nil)
}

// setupHugepages creates a hugepage mount at the specified directory.
func (ed *emptyDir) setupHugepages(dir string) error {
	if ed.mounter == nil {
		return fmt.Errorf("memory storage requested, but mounter is nil")
	}
	if err := ed.setupDir(dir); err != nil {
		return err
	}
	// Make SetUp idempotent.
	medium, isMnt, mountPageSize, err := ed.mountDetector.GetMountMedium(dir, ed.medium)
	klog.V(3).Infof("pod %v: setupHugepages: medium: %s, isMnt: %v, dir: %s, err: %v", ed.pod.UID, medium, isMnt, dir, err)
	if err != nil {
		return err
	}
	// If the directory is a mountpoint with medium hugepages of the same page size,
	// there is no work to do since we are already in the desired state.
	if isMnt && v1helper.IsHugePageMedium(medium) {
		// Medium is: Hugepages
		if ed.medium == v1.StorageMediumHugePages {
			return nil
		}
		if mountPageSize == nil {
			return fmt.Errorf("pod %v: mounted dir %s pagesize is not determined", ed.pod.UID, dir)
		}
		// Medium is: Hugepages-<size>
		// Mounted page size and medium size must be equal
		mediumSize, err := v1helper.HugePageSizeFromMedium(ed.medium)
		if err != nil {
			return err
		}
		if mountPageSize == nil || mediumSize.Cmp(*mountPageSize) != 0 {
			return fmt.Errorf("pod %v: mounted dir %s pagesize '%s' and requested medium size '%s' differ", ed.pod.UID, dir, mountPageSize.String(), mediumSize.String())
		}
		return nil
	}

	pageSizeMountOption, err := getPageSizeMountOption(ed.medium, ed.pod)
	if err != nil {
		return err
	}

	klog.V(3).Infof("pod %v: mounting hugepages for volume %v", ed.pod.UID, ed.volName)
	return ed.mounter.MountSensitiveWithoutSystemd("nodev", dir, "hugetlbfs", []string{pageSizeMountOption}, nil)
}

```

## mptyDir.sizeLimit(Quantity)-string
sizeLimit是这个EmptyDir卷所需的本地存储总量。大小限制也适用于内存介质。内存介质EmptyDir的最大使用量将是这里指定的SizeLimit与pod中所有容器的内存限制之和之间的最小值。默认值为nil，这意味着限制未定义