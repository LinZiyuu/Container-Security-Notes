

syncPod
```go
// syncPod is the transaction script for the sync of a single pod (setting up)
// a pod. This method is reentrant and expected to converge a pod towards the
// desired state of the spec. The reverse (teardown) is handled in
// syncTerminatingPod and syncTerminatedPod. If syncPod exits without error,
// then the pod runtime state is in sync with the desired configuration state
// (pod is running). If syncPod exits with a transient error, the next
// invocation of syncPod is expected to make progress towards reaching the
// runtime state. syncPod exits with isTerminal when the pod was detected to
// have reached a terminal lifecycle phase due to container exits (for
// RestartNever or RestartOnFailure) and the next method invoked will by
// syncTerminatingPod.
//
// Arguments:
//
// updateType - whether this is a create (first time) or an update, should
//   only be used for metrics since this method must be reentrant
// pod - the pod that is being set up
// mirrorPod - the mirror pod known to the kubelet for this pod, if any
// podStatus - the most recent pod status observed for this pod which can
//   be used to determine the set of actions that should be taken during
//   this loop of syncPod
//
// The workflow is:
// * If the pod is being created, record pod worker start latency
// * Call generateAPIPodStatus to prepare an v1.PodStatus for the pod
// * If the pod is being seen as running for the first time, record pod
//   start latency
// * Update the status of the pod in the status manager
// * Stop the pod's containers if it should not be running due to soft
//   admission
// * Ensure any background tracking for a runnable pod is started
// * Create a mirror pod if the pod is a static pod, and does not
//   already have a mirror pod
// * Create the data directories for the pod if they do not exist
// * Wait for volumes to attach/mount
// * Fetch the pull secrets for the pod
// * Call the container runtime's SyncPod callback
// * Update the traffic shaping for the pod's ingress and egress limits
//
// If any step of this workflow errors, the error is returned, and is repeated
// on the next syncPod call.
//
// This operation writes all events that are dispatched in order to provide
// the most accurate information possible about an error situation to aid debugging.
// Callers should not write an event if this operation returns an error.
func (kl *Kubelet) syncPod(ctx context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) {
	klog.V(4).InfoS("syncPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer func() {
		klog.V(4).InfoS("syncPod exit", "pod", klog.KObj(pod), "podUID", pod.UID, "isTerminal", isTerminal)
	}()

	// Latency measurements for the main workflow are relative to the
	// first time the pod was seen by kubelet.
	var firstSeenTime time.Time
	if firstSeenTimeStr, ok := pod.Annotations[kubetypes.ConfigFirstSeenAnnotationKey]; ok {
		firstSeenTime = kubetypes.ConvertToTimestamp(firstSeenTimeStr).Get()
	}

	// Record pod worker start latency if being created
	// TODO: make pod workers record their own latencies
	if updateType == kubetypes.SyncPodCreate {
		if !firstSeenTime.IsZero() {
			// This is the first time we are syncing the pod. Record the latency
			// since kubelet first saw the pod if firstSeenTime is set.
			metrics.PodWorkerStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
		} else {
			klog.V(3).InfoS("First seen time not recorded for pod",
				"podUID", pod.UID,
				"pod", klog.KObj(pod))
		}
	}

	// Generate final API pod status with pod and status manager status
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)
	// The pod IP may be changed in generateAPIPodStatus if the pod is using host network. (See #24576)
	// TODO(random-liu): After writing pod spec into container labels, check whether pod is using host network, and
	// set pod IP to hostIP directly in runtime.GetPodStatus
	podStatus.IPs = make([]string, 0, len(apiPodStatus.PodIPs))
	for _, ipInfo := range apiPodStatus.PodIPs {
		podStatus.IPs = append(podStatus.IPs, ipInfo.IP)
	}
	if len(podStatus.IPs) == 0 && len(apiPodStatus.PodIP) > 0 {
		podStatus.IPs = []string{apiPodStatus.PodIP}
	}

	// If the pod is terminal, we don't need to continue to setup the pod
	if apiPodStatus.Phase == v1.PodSucceeded || apiPodStatus.Phase == v1.PodFailed {
		kl.statusManager.SetPodStatus(pod, apiPodStatus)
		isTerminal = true
		return isTerminal, nil
	}

	// If the pod should not be running, we request the pod's containers be stopped. This is not the same
	// as termination (we want to stop the pod, but potentially restart it later if soft admission allows
	// it later). Set the status and phase appropriately
	runnable := kl.canRunPod(pod)
	if !runnable.Admit {
		// Pod is not runnable; and update the Pod and Container statuses to why.
		if apiPodStatus.Phase != v1.PodFailed && apiPodStatus.Phase != v1.PodSucceeded {
			apiPodStatus.Phase = v1.PodPending
		}
		apiPodStatus.Reason = runnable.Reason
		apiPodStatus.Message = runnable.Message
		// Waiting containers are not creating.
		const waitingReason = "Blocked"
		for _, cs := range apiPodStatus.InitContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
		for _, cs := range apiPodStatus.ContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
	}

	// Record the time it takes for the pod to become running
	// since kubelet first saw the pod if firstSeenTime is set.
	existingStatus, ok := kl.statusManager.GetPodStatus(pod.UID)
	if !ok || existingStatus.Phase == v1.PodPending && apiPodStatus.Phase == v1.PodRunning &&
		!firstSeenTime.IsZero() {
		metrics.PodStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
	}

	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// Pods that are not runnable must be stopped - return a typed error to the pod worker
	if !runnable.Admit {
		klog.V(2).InfoS("Pod is not runnable and must have running containers stopped", "pod", klog.KObj(pod), "podUID", pod.UID, "message", runnable.Message)
		var syncErr error
		p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
		if err := kl.killPod(pod, p, nil); err != nil {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
			syncErr = fmt.Errorf("error killing pod: %v", err)
			utilruntime.HandleError(syncErr)
		} else {
			// There was no error killing the pod, but the pod cannot be run.
			// Return an error to signal that the sync loop should back off.
			syncErr = fmt.Errorf("pod cannot be run: %s", runnable.Message)
		}
		return false, syncErr
	}

	// If the network plugin is not ready, only start the pod if it uses the host network
	if err := kl.runtimeState.networkErrors(); err != nil && !kubecontainer.IsHostNetworkPod(pod) {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.NetworkNotReady, "%s: %v", NetworkNotReadyErrorMsg, err)
		return false, fmt.Errorf("%s: %v", NetworkNotReadyErrorMsg, err)
	}

	// ensure the kubelet knows about referenced secrets or configmaps used by the pod
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		if kl.secretManager != nil {
			kl.secretManager.RegisterPod(pod)
		}
		if kl.configMapManager != nil {
			kl.configMapManager.RegisterPod(pod)
		}
	}

	// Create Cgroups for the pod and apply resource parameters
	// to them if cgroups-per-qos flag is enabled.
	pcm := kl.containerManager.NewPodContainerManager()
	// If pod has already been terminated then we need not create
	// or update the pod's cgroup
	// TODO: once context cancellation is added this check can be removed
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		// When the kubelet is restarted with the cgroups-per-qos
		// flag enabled, all the pod's running containers
		// should be killed intermittently and brought back up
		// under the qos cgroup hierarchy.
		// Check if this is the pod's first sync
		firstSync := true
		for _, containerStatus := range apiPodStatus.ContainerStatuses {
			if containerStatus.State.Running != nil {
				firstSync = false
				break
			}
		}
		// Don't kill containers in pod if pod's cgroups already
		// exists or the pod is running for the first time
		podKilled := false
		if !pcm.Exists(pod) && !firstSync {
			p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
			if err := kl.killPod(pod, p, nil); err == nil {
				podKilled = true
			} else {
				klog.ErrorS(err, "KillPod failed", "pod", klog.KObj(pod), "podStatus", podStatus)
			}
		}
		// Create and Update pod's Cgroups
		// Don't create cgroups for run once pod if it was killed above
		// The current policy is not to restart the run once pods when
		// the kubelet is restarted with the new flag as run once pods are
		// expected to run only once and if the kubelet is restarted then
		// they are not expected to run again.
		// We don't create and apply updates to cgroup if its a run once pod and was killed above
		if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
			if !pcm.Exists(pod) {
				if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
					klog.V(2).InfoS("Failed to update QoS cgroups while syncing pod", "pod", klog.KObj(pod), "err", err)
				}
				if err := pcm.EnsureExists(pod); err != nil {
					kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToCreatePodContainer, "unable to ensure pod container exists: %v", err)
					return false, fmt.Errorf("failed to ensure that the pod: %v cgroups exist and are correctly applied: %v", pod.UID, err)
				}
			}
		}
	}

	// Create Mirror Pod for Static Pod if it doesn't already exist
	if kubetypes.IsStaticPod(pod) {
		deleted := false
		if mirrorPod != nil {
			if mirrorPod.DeletionTimestamp != nil || !kl.podManager.IsMirrorPodOf(mirrorPod, pod) {
				// The mirror pod is semantically different from the static pod. Remove
				// it. The mirror pod will get recreated later.
				klog.InfoS("Trying to delete pod", "pod", klog.KObj(pod), "podUID", mirrorPod.ObjectMeta.UID)
				podFullName := kubecontainer.GetPodFullName(pod)
				var err error
				deleted, err = kl.podManager.DeleteMirrorPod(podFullName, &mirrorPod.ObjectMeta.UID)
				if deleted {
					klog.InfoS("Deleted mirror pod because it is outdated", "pod", klog.KObj(mirrorPod))
				} else if err != nil {
					klog.ErrorS(err, "Failed deleting mirror pod", "pod", klog.KObj(mirrorPod))
				}
			}
		}
		if mirrorPod == nil || deleted {
			node, err := kl.GetNode()
			if err != nil || node.DeletionTimestamp != nil {
				klog.V(4).InfoS("No need to create a mirror pod, since node has been removed from the cluster", "node", klog.KRef("", string(kl.nodeName)))
			} else {
				klog.V(4).InfoS("Creating a mirror pod for static pod", "pod", klog.KObj(pod))
				if err := kl.podManager.CreateMirrorPod(pod); err != nil {
					klog.ErrorS(err, "Failed creating a mirror pod for", "pod", klog.KObj(pod))
				}
			}
		}
	}

	// Make data directories for the pod
	if err := kl.makePodDataDirs(pod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToMakePodDataDirectories, "error making pod data directories: %v", err)
		klog.ErrorS(err, "Unable to make pod data directories for pod", "pod", klog.KObj(pod))
		return false, err
	}

	// Volume manager will not mount volumes for terminating pods
	// TODO: once context cancellation is added this check can be removed
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		// Wait for volumes to attach/mount
		if err := kl.volumeManager.WaitForAttachAndMount(pod); err != nil {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to attach or mount volumes: %v", err)
			klog.ErrorS(err, "Unable to attach or mount volumes for pod; skipping pod", "pod", klog.KObj(pod))
			return false, err
		}
	}

	// Fetch the pull secrets for the pod
	pullSecrets := kl.getPullSecretsForPod(pod)

	// Ensure the pod is being probed
	kl.probeManager.AddPod(pod)

	// Call the container runtime's SyncPod callback
	result := kl.containerRuntime.SyncPod(pod, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	if err := result.Error(); err != nil {
		// Do not return error if the only failures were pods in backoff
		for _, r := range result.SyncResults {
			if r.Error != kubecontainer.ErrCrashLoopBackOff && r.Error != images.ErrImagePullBackOff {
				// Do not record an event here, as we keep all event logging for sync pod failures
				// local to container runtime, so we get better errors.
				return false, err
			}
		}

		return false, nil
	}

	return false, nil
}
```


WaitForAttachAndMount
```go
func (vm *volumeManager) WaitForAttachAndMount(pod *v1.Pod) error {
	if pod == nil {
		return nil
	}

	expectedVolumes := getExpectedVolumes(pod)
	if len(expectedVolumes) == 0 {
		// No volumes to verify
		return nil
	}

	klog.V(3).InfoS("Waiting for volumes to attach and mount for pod", "pod", klog.KObj(pod))
	uniquePodName := util.GetUniquePodName(pod)

	// Some pods expect to have Setup called over and over again to update.
	// Remount plugins for which this is true. (Atomically updating volumes,
	// like Downward API, depend on this to update the contents of the volume).
	vm.desiredStateOfWorldPopulator.ReprocessPod(uniquePodName)

	err := wait.PollImmediate(
		podAttachAndMountRetryInterval,
		podAttachAndMountTimeout,
		vm.verifyVolumesMountedFunc(uniquePodName, expectedVolumes))

	if err != nil {
		unmountedVolumes :=
			vm.getUnmountedVolumes(uniquePodName, expectedVolumes)
		// Also get unattached volumes for error message
		unattachedVolumes :=
			vm.getUnattachedVolumes(expectedVolumes)

		if len(unmountedVolumes) == 0 {
			return nil
		}

		return fmt.Errorf(
			"unmounted volumes=%v, unattached volumes=%v: %s",
			unmountedVolumes,
			unattachedVolumes,
			err)
	}

	klog.V(3).InfoS("All volumes are attached and mounted for pod", "pod", klog.KObj(pod))
	return nil
}
```

```go
// verifyVolumesMountedFunc returns a method that returns true when all expected
// volumes are mounted.
func (vm *volumeManager) verifyVolumesMountedFunc(podName types.UniquePodName, expectedVolumes []string) wait.ConditionFunc {
	return func() (done bool, err error) {
		if errs := vm.desiredStateOfWorld.PopPodErrors(podName); len(errs) > 0 {
			return true, errors.New(strings.Join(errs, "; "))
		}
		return len(vm.getUnmountedVolumes(podName, expectedVolumes)) == 0, nil
	}
}
```



























