# docker镜像和containerd镜像


## 术语
#### cri
CRI即容器运行时接口（Container Runtime Interface），相关proto文件可以参考：k8s.io/cri-api/pkg/apis/runtime/v1alpha2/api.proto.
CRI定义了两种服务类型接口：RuntimeService和ImageService，这也意味着CRI服务需要实现两种功能：runtime的管理和image的管理。
##### RuntimeService
RuntimeService的定义如下，根据接口定义，可以看出来RuntimeService需要负责runtime的管理工作，包括sandbox、container的增删改查以及登录容器（exec）等功能
```
service RuntimeService {
    rpc Version(VersionRequest) returns (VersionResponse) {}
    rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse) {}
    rpc StopPodSandbox(StopPodSnadboxRequest) returns (StopPodSandboxResponse) {}
    rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse) {}
    rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse) {}
    rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse) {} 
    rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse) {}
    rpc StartContainer(StartContainerRequest) returns (StartContainerResponse) {}
    rpc StopContainer(StopContainerRequest) returns (StopContainerResponse) {}
    rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse) {}
    rpc ListContainers(ListContainersRequest) returns (ListContainersResponse) {}
    rpc UpdateContainerResources(UpdateContainerResourcesRequest) returns (UpdateContainerResourcesResponse) {}
    rpc ReopenContainerLog(ReopenContainerLogRequest) returns (ReopenContainerLogResponse) {}
    rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse) {}
    rpc Exec(ExecRequest) returns (ExecResponse) {}
    rpc Attach(AttachRequest) returns (AttachResponse) {}
    rpc PortForward(PortForwardRequest) returns (PortForwardResponse) {}
    rpc ContainerStats(ContainerStatsRequest) returns (ContainerStatsResponse) {}
    rpc ListContainerStats(ListContainerStatsRequest) returns (ListContainerStatsResponse) {}
    rpc UpdateRuntimeConfig(UpdateRuntimeConfigRequest) returns (UpdateRuntimeConfigResponse) {}
    rpc Status(StatusRequest) returns (StatusResponse) {}
}
```
##### ImageService
ImageService的定义如下，主要负责容器镜像的管理（注意并没有PushImage功能）：
```
service ImageService {
    rpc ListImages(ListImagesRequest) returns (ListImagesResponse) {}
    rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse) {}
    rpc PullImage(PullImageRequest) returns (PullImageResponse) {}
    rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse) {}
    rpc ImageFsInfo(ImageFsInfoRequest) returns (ImageFsInfoResponse) {}
}
```

#### docker
当前表述的docker包含docker和dockerd等一整套，是CRI的一种实现，即dockerd实现了CRI接口，提供了rpc接口给上层服务（如kubelet)调用，用于管理runtime和image。
docker操作image主要有以下命令：
```
// 获取镜像列表
# docker images
// 拉镜像
# docker pull {image}
// 推镜像
# docker push {image}
// 保存镜像为tar包
# docker save -o {image.tar.gz} {image}
// 从tar包载入镜像
# docker load < {image.tar.gz}
```

#### containerd
containerd是CRI的另一种实现，关于containerd和dockerd的历史渊源本文不展开叙述，在本文中把containerd简单理解为dockerd的功能即可。

#### crictl
crictl是标准CRI的客户端，通过rpc方式访问CRI实现（如dockerd或者containerd）。
crictl的项目代码在github.com/kubernetes-sigs/cri-tools，可看作由kubernetes团队维护管理。
既然是CRI的标准客户端，从上边ImageService服务定义可以看出，crictl是没有推送镜像（PushImage）功能的，但是crictl可以查看dockerd镜像和containerd镜像。

#### ctr
containerd的客户端，除了实现了CRI的客户端，还封装了一些其它功能，如events、task等。
ctr代码在github.com/containerd/containerd/cmd/ctr，归属于containerd团队维护管理。
通过ctr --help命令可以看到ctr支持--namespace value, -n value参数，containerd正是加入了命名空间来做镜像隔离，这一点是与dockerd不同的。
containerd镜像的命名空间是k8s.io，ctr查看containerd镜像的命令为：
```
# ctr -n k8s.io images ls
```

## docker镜像与containerd镜像相互转换
dockerd镜像和containerd镜像存储在不同的地方，docker命令不能查看containerd的镜像，ctr命令也不能查看dockerd的镜像。
ctr查看dockerd镜像会报如下错误：
```
# ctr --address /var/run/dockershim.sock images ls
ctr: failed to list images: unknown service containerd.services.images.v1.Images: not implemented
```
#### docker镜像转换成containerd镜像
```
# docker save -o {image.tar.gz} {image}
# ctr -n k8s.io images import {image.tar.gz}
```
#### containerd镜像转换成docker镜像
```
# ctr -n k8s.io images export {image.tar.gz} {image}
# docker load < {image.tar.gz}
```
