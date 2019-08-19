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
![1](../../image/kubernetes/csi1.png)   


具体代码待补充。。。。。