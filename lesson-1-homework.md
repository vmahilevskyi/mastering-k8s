## An issue with downloading cloud-controller-manager
A command `sudo curl -L "https://dl.k8s.io/v1.30.0/bin/linux/amd64/cloud-controller-manager" -o kubebuilder/bin/cloud-controller-manager` returns:
```xml
<?xml version='1.0' encoding='UTF-8'?><Error><Code>NoSuchKey</Code><Message>The specified key does not exist.</Message><Details>No such object: 767373bbdcb8270361b96548387bf2a9ad0d48758c35/release/v1.30.0/bin/linux/amd64/cloud-controller-manager</Details></Error>
```
I've double-checked and there is no `cloud-controller-manager` binary at https://www.downloadkubernetes.com/.
Maybe it's expected on this stage

## Verifications
### Check node status
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (lesson-1) $ sudo kubebuilder/bin/kubectl get nodes
NAME                STATUS   ROLES    AGE   VERSION
codespaces-b7225a   Ready    master   11m   v1.30.0
```

### Check component status
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (lesson-1) $ sudo kubebuilder/bin/kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
etcd-0               Healthy   ok        
controller-manager   Healthy   ok        
scheduler            Healthy   ok        
```

### Check API server health
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (lesson-1) $ sudo kubebuilder/bin/kubectl get --raw='/readyz?verbose'
[+]ping ok
[+]log ok
[+]etcd ok
[+]etcd-readiness ok
[+]informer-sync ok
[+]poststarthook/start-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/max-in-flight-filter ok
[+]poststarthook/storage-object-count-tracker-hook ok
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/start-service-ip-repair-controllers ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-system-namespaces-controller ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/start-kube-apiserver-identity-lease-controller ok
[+]poststarthook/start-kube-apiserver-identity-lease-garbage-collector ok
[+]poststarthook/start-legacy-token-tracking-controller ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/apiservice-discovery-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
[+]poststarthook/apiservice-openapiv3-controller ok
[+]shutdown ok
readyz check passed
```

### Create Deployment 
`sudo  kubebuilder/bin/kubectl create deploy demo --image nginx`

Pod scheduling was failing with the following reason:
```
I1004 16:19:29.529553   46541 schedule_one.go:1046] "Unable to schedule pod; no fit; waiting" pod="default/demo-677cfb9d49-k2kwm" err="0/1 nodes are available: 1 node(s) had untolerated taint {node.cloudprovider.kubernetes.io/uninitialized: true}. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling."
```
Fixed by adding proper `tolerations` to `spec.template.spec` in `demo` Deployment via `kubectl edit`.
```
tolerations:
- effect: NoSchedule
  key: node.cloudprovider.kubernetes.io/uninitialized
  operator: Equal
  value: "true"
```

### Check all resources
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (lesson-1) $ sudo kubebuilder/bin/kubectl get all -A
NAMESPACE   NAME                        READY   STATUS    RESTARTS   AGE
default     pod/demo-7586d747b4-bfsq7   1/1     Running   0          8m37s

NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   39m

NAMESPACE   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
default     deployment.apps/demo   1/1     1            1           14m

NAMESPACE   NAME                              DESIRED   CURRENT   READY   AGE
default     replicaset.apps/demo-677cfb9d49   0         0         0       14m
default     replicaset.apps/demo-7586d747b4   1         1         1       8m37s
```
