# k8s资源分类

## 一、wordflow类

- pods
- replicasets/rs
- daemonsets/ds
- deployments/deploy
- statefulsets/sts
- cronjobs/cj
- jobs

## 二、网络负载类

- endpoints/ep
- ingresses/ing
- ingressclasses
- services/svc
- networkpolicies/netpol

## 三、存储配置类

namespace级

- persistentvolumeclaims/pvc
- configmaps/cm
- secrets
- volumesnapshots

集群级

- storageclasses/sc
- persistentvolumes/pv

## 四、权限类

namespace级

- bindings
- serviceaccounts/sa
- rolebindings
- roles

集群级

- role
- clusterrole
- roleBinding
- clusterrolebinding

## 五、集群管理类

- namespace/ns
- node/no
- customresourcedefinitions/crd
- apiservices
- hpa
- events/ev
- limitranges/limits
