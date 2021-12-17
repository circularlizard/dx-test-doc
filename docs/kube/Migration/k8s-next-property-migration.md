---
id: k8s-next-property-migration
title: Operator property migration
---

## Introduction

In order to migrate from Operator based deployments to Helm, it might be necessary that you migrate the deployment configuration of your old Operator deployment first.
This page will provide you with a mapping that allows you to re-use values from your old `deployment.properties` file in your new `custom-values.yaml`.

## Preparation

Ensure that you have followed the preparation process for a fresh Helm deployment, including creation of your `custom-values.yaml`.  

You will need the properties file that you used with DXCTL to perform your old Operator deployment. If you do not have that property file at hand, you can refer to the DXCTL documentation on how to use the `getProperties` function of DXCTL to extract a properties file from your existing deployment.

## Property mappings

This chapter will go through relevant properties that can be found in a DXCTL property file and how they may be mapped to the new `custom-values.yaml`.

**Please note: You should only transfer settings that you have adjusted for your Operator deployment. It is not recommended to overwrite all Helm defaults with the defaults of the old Operator deployment. Only migrate settings that are relevant for you or have been adjusted by you prior deploying the Operator with DXCTL.**

### General properties

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| dx.namespace | n/a | Namespace used for the deployment, will be handed to helm directly via CLI |
| dx.name | n/a | Deployment name, will be handed to helm directly via CLI |
| dx.pullpolicy | images.pullPolicy | Determines the image pull policy for all container images |
| dx.pod.nodeselector | nodeSelector.* | NodeSelector used for Pods, can now be done per application in Helm |
| dx.config.authoring | configuration.core.tuning.authoring | Selects if the instance should be tuned for authoring or not |
| composer.enabled | applications.contentComposer | Selects if Content Composer will be deployed or not |
| dam.enabled | applications.digitalAssetManagement | Selects if Digital Asset Management will be deployed or not |
| persist.force-read | n/a | Read-only fallback enablement, always enabled in Helm |

### Storage properties

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| dx.volume | volumes.core.profile.volumeName | Name of the volume to be used for DX Core Profile |
| dx.volume.size | volumes.core.profile.requests.storage | Size of the volume to be used for DX Core Profile |
| dx.storageclass | volumes.core.profile.storageClassName | StorageClass of the volume used for DX Core Profile |
| dx.splitlogging: false | n/a | Determines if log directory uses a separate volume, always enabled in Helm |
| dx.logging.stgclass | volumes.core.log.storageClassName | StorageClass for DX Core logging volume |
| dx.logging.size | volumes.core.log.requests.storage | StorageClass for DX Core logging volume |
| dx.tranlogging | n/a | Determines if the transaction log directory uses a separate volume, always enabled in Helm |
| dx.tranlogging.reclaim | n/a | Reclaimpolicy for DX Core transaction log volume. Determined by PV instead of Helm |
| dx.tranlogging.stgclass | volumes.core.tranlog.storageClassName | StorageClass for the DX Core transaction log volume |
| dx.tranlogging.size | volumes.core.tranlog.requests.storage | Size used for the DX Core transaction log volume |
| remote.search.volume | volumes.remoteSearch.prsprofile.volumeName | Name of the volume used for the DX Remote Search profile |
| remote.search.stgclass | volumes.remoteSearch.prsprofile.storageClassName | StorageClass of the volume for the DX Remote Search profile |
| dam.volume | volumes.digitalAssetManagement.binaries.volumeName | Name of the volume used for DAM |
| dam.stgclass | volumes.digitalAssetManagement.binaries.storageClassName | StorageClass of the volume used for DAM |

### Networking properties

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| dx.path.contextroot | networking.core.contextRoot | Context root used for DX |
| dx.path.personalized | networking.core.personalizedHome | Personalized URL path for DX |
| dx.path.home | networking.core.home | Non personalized URL path for DX |
| dx.deploy.host.override | networking.core.host | Hostname that should be used instead of the Loadbalancer hostname |
| dx.deploy.host.override.force | n/a | Force the use of the override host, obsolete in helm |
| dx.config.cors | networking.addon.*.corsOrigin | CORS configuration for applications, can be configured per application in Helm |
| hybrid.enabled | n/a | Determines if hybrid is enabled or not, Helm will derive this from other networking and application settings |
| hybrid.url | networking.core.host | URL of the DX Core instance in a hybrid deployment |
| hybrid.port | networking.core.port | Port of the DX Core instance in a hybrid deployment |

### Scaling properties

#### Core

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| dx.minreplicas | scaling.horizontalPodAutoScaler.core.minReplicas | Minimum amount of Pods when scaling is enabled |
| dx.maxreplicas | scaling.horizontalPodAutoScaler.core.maxReplicas | Maximum amount of Pods when scaling is enabled |
| dx.replicas | scaling.replicas.core | Default amount of Pods when scaling is disabled |
| dx.targetcpuutilizationpercent | scaling.horizontalPodAutoScaler.core.targetCPUUtilizationPercentage | CPU Target for autoscaling |
| dx.targetmemoryutilizationpercent | scaling.horizontalPodAutoScaler.core.targetMemoryUtilizationPercentage | Memory Target for autoscaling |

#### Ring API

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| api.targetcpuutilizationpercent | scaling.horizontalPodAutoScaler.ringApi.targetCPUUtilizationPercentage | CPU Target for autoscaling |
| api.targetmemoryutilizationpercent | scaling.horizontalPodAutoScaler.ringApi.targetMemoryUtilizationPercentage | Memory Target for autoscaling |
| api.minreplicas | scaling.horizontalPodAutoScaler.ringApi.minReplicas | Minimum amount of Pods when scaling is enabled |
| api.maxreplicas | scaling.horizontalPodAutoScaler.ringApi.maxReplicas | Maximum amount of Pods when scaling is enabled |

#### Content Composer

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| composer.targetcpuutilizationpercent | scaling.horizontalPodAutoScaler.contentComposer.targetCPUUtilizationPercentage | CPU Target for autoscaling |
| composer.targetmemoryutilizationpercent | scaling.horizontalPodAutoScaler.contentComposer.targetMemoryUtilizationPercentage | Memory Target for autoscaling |
| composer.minreplicas | scaling.horizontalPodAutoScaler.contentComposer.minReplicas | Minimum amount of Pods when scaling is enabled |
| composer.maxreplicas | scaling.horizontalPodAutoScaler.contentComposer.maxReplicas | Maximum amount of Pods when scaling is enabled |

#### Digital Asset Management

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| dam.targetcpuutilizationpercent | scaling.horizontalPodAutoScaler.digitalAssetManagement.targetCPUUtilizationPercentage | CPU Target for autoscaling |
| dam.targetmemoryutilizationpercent | scaling.horizontalPodAutoScaler.digitalAssetManagement.targetMemoryUtilizationPercentage | Memory Target for autoscaling |
| dam.minreplicas | scaling.horizontalPodAutoScaler.digitalAssetManagement.minReplicas | Minimum amount of Pods when scaling is enabled |
| dam.maxreplicas | scaling.horizontalPodAutoScaler.digitalAssetManagement.maxReplicas | Maximum amount of Pods when scaling is enabled |

#### Image Processor

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| imgproc.targetcpuutilizationpercent | scaling.horizontalPodAutoScaler.imageProcessor.targetCPUUtilizationPercentage | CPU Target for autoscaling |
| imgproc.targetmemoryutilizationpercent | scaling.horizontalPodAutoScaler.imageProcessor.targetMemoryUtilizationPercentage | Memory Target for autoscaling |
| imgproc.minreplicas | scaling.horizontalPodAutoScaler.imageProcessor.minReplicas | Minimum amount of Pods when scaling is enabled |
| imgproc.maxreplicas | scaling.horizontalPodAutoScaler.imageProcessor.maxReplicas | Maximum amount of Pods when scaling is enabled |

### Resource allocation properties

#### Core

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| dx.request.cpu | resources.core.requests.cpu | CPU Request |
| dx.request.memory | resources.core.requests.memory | Memory Request |
| dx.limit.cpu | resources.core.limits.cpu | CPU Limit |
| dx.limit.memory | resources.core.limits.memory | Memory Limit |

#### Ring API

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| api.request.cpu | resources.ringApi.requests.cpu | CPU Request |
| api.request.memory | resources.ringApi.requests.memory | Memory Request |
| api.limit.cpu | resources.ringApi.limits.cpu | CPU Limit |
| api.limit.memory | resources.ringApi.limits.memory | Memory Limit |

#### Content Composer

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| composer.request.cpu | resources.contentComposer.requests.cpu | CPU Request |
| composer.request.memory | resources.contentComposer.requests.memory | Memory Request |
| composer.limit.cpu | resources.contentComposer.limits.cpu | CPU Limit |
| composer.limit.memory | resources.contentComposer.limits.memory | Memory Limit |

#### Digital Asset Management

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| dam.request.cpu | resources.digitalAssetManagement.requests.cpu | CPU Request |
| dam.request.memory | resources.digitalAssetManagement.requests.memory | Memory Request |
| dam.limit.cpu | resources.digitalAssetManagement.limits.cpu | CPU Limit |
| dam.limit.memory | resources.digitalAssetManagement.limits.memory | Memory Limit |

#### Persistence

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| persist.request.cpu | resources.persistence.requests.cpu | CPU Request |
| persist.request.memory | resources.persistence.requests.memory | Memory Request |
| persist.limit.cpu | resources.persistence.limits.cpu | CPU Limit |
| persist.limit.memory | resources.persistence.limits.memory | Memory Limit |

#### Image Processor

| DXCTL property | values.yaml key | Description |
| -- | -- | -- |
| imgproc.request.cpu | resources.imageProcessor.requests.cpu | CPU Request |
| imgproc.request.memory | resources.imageProcessor.requests.memory | Memory Request |
| imgproc.limit.cpu | resources.imageProcessor.limits.cpu | CPU Limit |
| imgproc.limit.memory | resources.imageProcessor.limits.memory | Memory Limit |
