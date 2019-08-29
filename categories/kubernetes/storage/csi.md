# CSI(Container Storage Interface)
CSI是容器存储接口规范，实现CSI的插件可以在容器编排（CO）系统中如：Cloud Foundry，Kubernetes，Mesos中工作。

## CSI接口
Identity主要是注册driverName等，然后storageClass中的Provisioner就是通过这个名词找到具体调用哪个CSI插件。    
GetPluginCapabilities主要告诉k8s当前CSI不具备哪个能力（比如不需要provision，attach等）。  
probe健康检查接口。  
```go
type IdentityServer interface {
	GetPluginInfo(context.Context, *GetPluginInfoRequest) (*GetPluginInfoResponse, error)
	GetPluginCapabilities(context.Context, *GetPluginCapabilitiesRequest) (*GetPluginCapabilitiesResponse, error)
	Probe(context.Context, *ProbeRequest) (*ProbeResponse, error)
}
```

Controller主要提供provesion阶段，包括CreateVolume，DeleteVolume，  
还有Attach阶段的ControllerPublishVolume，ControllerUnpublishVolume。
```go
type ControllerServer interface {
	CreateVolume(context.Context, *CreateVolumeRequest) (*CreateVolumeResponse, error)
	DeleteVolume(context.Context, *DeleteVolumeRequest) (*DeleteVolumeResponse, error)
	ControllerPublishVolume(context.Context, *ControllerPublishVolumeRequest) (*ControllerPublishVolumeResponse, error)
	ControllerUnpublishVolume(context.Context, *ControllerUnpublishVolumeRequest) (*ControllerUnpublishVolumeResponse, error)
	ValidateVolumeCapabilities(context.Context, *ValidateVolumeCapabilitiesRequest) (*ValidateVolumeCapabilitiesResponse, error)
	ListVolumes(context.Context, *ListVolumesRequest) (*ListVolumesResponse, error)
	GetCapacity(context.Context, *GetCapacityRequest) (*GetCapacityResponse, error)
	ControllerGetCapabilities(context.Context, *ControllerGetCapabilitiesRequest) (*ControllerGetCapabilitiesResponse, error)
	CreateSnapshot(context.Context, *CreateSnapshotRequest) (*CreateSnapshotResponse, error)
	DeleteSnapshot(context.Context, *DeleteSnapshotRequest) (*DeleteSnapshotResponse, error)
	ListSnapshots(context.Context, *ListSnapshotsRequest) (*ListSnapshotsResponse, error)
}
```

node接口主要是管理mount阶段，需要在具体pod的调度node上操作，一般以DaemonSet部署，  
具体包括NodeStageVolume（格式化，挂载），NodePublishVolume（绑定挂载）等。
```go
type NodeServer interface {
	NodeStageVolume(context.Context, *NodeStageVolumeRequest) (*NodeStageVolumeResponse, error)
	NodeUnstageVolume(context.Context, *NodeUnstageVolumeRequest) (*NodeUnstageVolumeResponse, error)
	NodePublishVolume(context.Context, *NodePublishVolumeRequest) (*NodePublishVolumeResponse, error)
	NodeUnpublishVolume(context.Context, *NodeUnpublishVolumeRequest) (*NodeUnpublishVolumeResponse, error)
	// NodeGetId is being deprecated in favor of NodeGetInfo and will be
	// removed in CSI 1.0. Existing drivers, however, may depend on this
	// RPC call and hence this RPC call MUST be implemented by the CSI
	// plugin prior to v1.0.
	NodeGetId(context.Context, *NodeGetIdRequest) (*NodeGetIdResponse, error)
	NodeGetCapabilities(context.Context, *NodeGetCapabilitiesRequest) (*NodeGetCapabilitiesResponse, error)
	// Prior to CSI 1.0 - CSI plugins MUST implement both NodeGetId and
	// NodeGetInfo RPC calls.
	NodeGetInfo(context.Context, *NodeGetInfoRequest) (*NodeGetInfoResponse, error)
}
```

## External Components
External Components是grpc的client负责调用CSI插件的API，主要包括Driver Registrar，External Provisioner，External Attacher。  
Driver Registrar负责调用CSI的Identity向kubelet注册CSI插件。  因为kubelet不通过External Components调用CSI，所以需要CSI INFO。
External Provisioner主要watch APIServer的PVC对象，根据PVC然后调用CSI的Controller的provesion创建volume。  
External Attacher这个也太watch APIServer的VolumeAttachment API对象的变化，然后调用CSI的Controller的ControllerPublish进行Attach Volume操作。  

而Node的mount操作是由kubelet的VolumeManagerReconciler调用的。

流程图（网络图）：  
![1](../../../image/kubernetes/CSI1.png)   


## PVC创建流程(Dynamic Provisioning)
首先PersistentVolumeBinderController控制循环会去watch PVC的变化，但已有的volume没有符合条件的时候，
根据storageClass的Provisioner去找到plugin，如果不存在plugin或者Migrated的话，就由CSI处理，其实就是给PVC加上下面的Annotation。  
```go
AnnStorageProvisioner = "volume.beta.kubernetes.io/storage-provisioner"
```

关键函数：  
```go
controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
```
(ctrl *PersistentVolumeController) syncUnboundClaim
```go
func (ctrl *PersistentVolumeController) syncUnboundClaim(claim *v1.PersistentVolumeClaim) error {
	// This is a new PVC that has not completed binding
	// OBSERVATION: pvc is "Pending"
	if claim.Spec.VolumeName == "" {
		// User did not care which PV they get.
		delayBinding, err := pvutil.IsDelayBindingMode(claim, ctrl.classLister)
		if err != nil {
			return err
		}

		// [Unit test set 1]
		volume, err := ctrl.volumes.findBestMatchForClaim(claim, delayBinding)
		if err != nil {
			klog.V(2).Infof("synchronizing unbound PersistentVolumeClaim[%s]: Error finding PV for claim: %v", claimToClaimKey(claim), err)
			return fmt.Errorf("Error finding PV for claim %q: %v", claimToClaimKey(claim), err)
		}
		if volume == nil {
			klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: no volume found", claimToClaimKey(claim))
			// No PV could be found
			// OBSERVATION: pvc is "Pending", will retry
			switch {
			case delayBinding && !pvutil.IsDelayBindingProvisioning(claim):
				ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.WaitForFirstConsumer, "waiting for first consumer to be created before binding")
			case v1helper.GetPersistentVolumeClaimClass(claim) != "":
				if err = ctrl.provisionClaim(claim); err != nil {
					return err
				}
```
```go
func (ctrl *PersistentVolumeController) provisionClaim(claim *v1.PersistentVolumeClaim) error {
	if !ctrl.enableDynamicProvisioning {
		return nil
	}
	klog.V(4).Infof("provisionClaim[%s]: started", claimToClaimKey(claim))
	opName := fmt.Sprintf("provision-%s[%s]", claimToClaimKey(claim), string(claim.UID))
	plugin, storageClass, err := ctrl.findProvisionablePlugin(claim)
	if err != nil {
		ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.ProvisioningFailed, err.Error())
		klog.Errorf("error finding provisioning plugin for claim %s: %v", claimToClaimKey(claim), err)
		// failed to find the requested provisioning plugin, directly return err for now.
		// controller will retry the provisioning in every syncUnboundClaim() call
		// retain the original behavior of returning nil from provisionClaim call
		return nil
	}
	ctrl.scheduleOperation(opName, func() error {
		// create a start timestamp entry in cache for provision operation if no one exists with
		// key = claimKey, pluginName = provisionerName, operation = "provision"
		claimKey := claimToClaimKey(claim)
		ctrl.operationTimestamps.AddIfNotExist(claimKey, ctrl.getProvisionerName(plugin, storageClass), "provision")
		var err error
		if plugin == nil || plugin.IsMigratedToCSI() {
			_, err = ctrl.provisionClaimOperationExternal(claim, plugin, storageClass)
		} else {
			_, err = ctrl.provisionClaimOperation(claim, plugin, storageClass)
		}
```
```go
func (ctrl *PersistentVolumeController) provisionClaimOperationExternal(
	claim *v1.PersistentVolumeClaim,
	plugin vol.ProvisionableVolumePlugin,
	storageClass *storage.StorageClass) (string, error) {
	claimClass := v1helper.GetPersistentVolumeClaimClass(claim)
	klog.V(4).Infof("provisionClaimOperationExternal [%s] started, class: %q", claimToClaimKey(claim), claimClass)
	// Set provisionerName to external provisioner name by setClaimProvisioner
	var err error
	provisionerName := storageClass.Provisioner
	if plugin != nil {
		// update the provisioner name to use the CSI in-tree name
		provisionerName, err = ctrl.getCSINameFromIntreeName(storageClass.Provisioner)
		if err != nil {
			strerr := fmt.Sprintf("error getting CSI name for In tree plugin %s: %v", storageClass.Provisioner, err)
			klog.V(2).Infof("%s", strerr)
			ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.ProvisioningFailed, strerr)
			return provisionerName, err
		}
	}
	// Add provisioner annotation so external provisioners know when to start
	newClaim, err := ctrl.setClaimProvisioner(claim, provisionerName)
	if err != nil {
```
```go
func (ctrl *PersistentVolumeController) setClaimProvisioner(claim *v1.PersistentVolumeClaim, provisionerName string) (*v1.PersistentVolumeClaim, error) {
	if val, ok := claim.Annotations[pvutil.AnnStorageProvisioner]; ok && val == provisionerName {
		// annotation is already set, nothing to do
		return claim, nil
	}

	// The volume from method args can be pointing to watcher cache. We must not
	// modify these, therefore create a copy.
	claimClone := claim.DeepCopy()
	metav1.SetMetaDataAnnotation(&claimClone.ObjectMeta, pvutil.AnnStorageProvisioner, provisionerName)
	newClaim, err := ctrl.kubeClient.CoreV1().PersistentVolumeClaims(claim.Namespace).Update(claimClone)
	if err != nil {
		return newClaim, err
	}
	_, err = ctrl.storeClaimUpdate(newClaim)
	if err != nil {
		return newClaim, err
	}
	return newClaim, nil
}
```

```go
ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.ExternalProvisioning, msg)
```

然后externla-provisioner watch到event，通过grpc调用CSI ControllerServer的CreateVolume创建volume。  
externla-provisioner代码：[LINK](https://github.com/kubernetes-csi/external-provisioner)  


接着是startAttachDetachController控制器循环，检查pod的pv是否已经和node挂载，如果未挂载就创建VolumeAttachment  
```go
controllers["attachdetach"] = startAttachDetachController  
```

然后就是external-attacher watch VolumeAttachment API对象的变化，去调用CSI ControllerServer的ControllerPublishVolume进程进行attach操作  
external-attacher代码：[LINK](https://github.com/kubernetes-csi/external-attacher)   

最后kubelet的VolumeManagerReconciler控制循环调用CSI Node完成volume的mount操作。