# What happens when ... Kubernetes edition!

```bash
kubectl run --image=nginx --replicas=3
```

## contents

1. [kubectl](#kubectl)
   - [validation and generators](#validation-and-generators)
   - [API groups and version negotiation](#api-groups-and-version-negotiation)
   - [client-side auth](#client-auth)
1. [kube-apiserver](#kube-apiserver)
   - [authentication](#authentication)
   - [authorization](#authorization)
   - [admission control](#admission-control)
1. [etcd](#etcd)
1. [initializers](#initializers)
1. [control loops](#control-loops)
   - [deployments controller](#deployments-controller)
   - [replicasets controller](#replicasets-controller)
   - [informers](#informers)
   - [scheduler](#scheduler)
1. [kubelet](#kubelet)
   - [pod sync](#pod-sync)
   - [CRI and pause containers](#cri-and-pause-containers)
   - [CNI and pod networking](#cni-and-pod-networking)
   - [inter-host networking](#inter-host-networking)
   - [container startup](#container-startup)
1. [wrap-up](#wrap-up)

## kubectl

### Validation and generators



### API groups and version negotiation



### Client auth



## kube-apiserver

### Authentication



### Authorization



### Admission control



## etcd



## Initializers



```
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```



## Control loops

### Deployments controller



### ReplicaSets controller



### Informers



### Scheduler



## kubelet 

### Pod sync



### CRI and pause containers



### CNI and pod networking



```yaml
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```



### Inter-host networking



### Container startup



## Wrap-up

