# Steps
## Start containerd
```
export PATH=$PATH:/opt/cni/bin:kubebuilder/bin
sudo PATH=$PATH:/opt/cni/bin:/usr/sbin /opt/cni/bin/containerd -c /etc/containerd/config.toml &
```
## Start kubelet
This time I've moved most of the arguments into the `/var/lib/kubelet/config.yaml`. 
I've referenced the manifest from the video lecture.
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: true
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubelet/ca.crt"
authorization:
  mode: AlwaysAllow
clusterDomain: "cluster.local"
clusterDNS:
  - "10.0.0.10"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
failSwapOn: false
seccompDefault: true
serverTLSBootstrap: false
containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
staticPodPath: "/etc/kubernetes/manifests"
tlsCertFile: /var/lib/kubelet/pki/kubelet.crt
tlsPrivateKeyFile: /var/lib/kubelet/pki/kubelet.key
cgroupDriver: cgroupfs
maxPods: 5
providerID: ""
```
Starting kubelet
```
HOST_IP=$(hostname -I | awk '{print $1}')
sudo PATH=$PATH:/opt/cni/bin:/usr/sbin kubebuilder/bin/kubelet \
    --kubeconfig=/var/lib/kubelet/kubeconfig \
    --config=/var/lib/kubelet/config.yaml \
    --root-dir=/var/lib/kubelet \
    --cert-dir=/var/lib/kubelet/pki \
    --hostname-override=$(hostname) \
    --node-ip=$HOST_IP \
    --v=1 &
```
## Added manifests for staticPods
```
sudo cp control-plane-manifests/etcd.yaml /etc/kubernetes/manifests
sudo cp control-plane-manifests/kube-apiserver.yaml /etc/kubernetes/manifests
sudo cp control-plane-manifests/kube-scheduler.yaml /etc/kubernetes/manifests
sudo cp control-plane-manifests/kube-controller-manager.yaml /etc/kubernetes/manifests
```
## Start Deployment with 3 replicas
```
sudo kubebuilder/bin/kubectl create deploy demo --image nginx --replicas 3
```
Only 1 Pod is in Running state. Others are in Pending state 
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (lesson-2) $ sudo ./kubebuilder/bin/kubectl get pods  
NAME                    READY   STATUS    RESTARTS   AGE
demo-677cfb9d49-d9qg9   0/1     Pending   0          3m56s
demo-677cfb9d49-pcpc9   1/1     Running   0          3m56s
demo-677cfb9d49-pwrhs   0/1     Pending   0          3m56s
```
A reason could be found by `sudo ./kubebuilder/bin/kubectl describe pod/demo-677cfb9d49-d9qg9`:
```
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  2m58s  default-scheduler  0/1 nodes are available: 1 Too many pods. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
```
Adjusting the `maxPods` spec in `/var/lib/kubelet/config.yaml` fixed the issue.
## Verification
### Check all resources
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (lesson-2) $ sudo kubebuilder/bin/kubectl get all -A
NAMESPACE     NAME                                            READY   STATUS    RESTARTS      AGE
default       pod/demo-677cfb9d49-d9qg9                       1/1     Running   0             43m
default       pod/demo-677cfb9d49-pcpc9                       1/1     Running   0             43m
default       pod/demo-677cfb9d49-pwrhs                       1/1     Running   0             43m
kube-system   pod/etcd-codespaces-b7225a                      1/1     Running   0             62m
kube-system   pod/kube-apiserver-codespaces-b7225a            1/1     Running   7 (76m ago)   63m
kube-system   pod/kube-controller-manager-codespaces-b7225a   1/1     Running   0             43m
kube-system   pod/kube-scheduler-codespaces-b7225a            1/1     Running   0             50m

NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   64m

NAMESPACE   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
default     deployment.apps/demo   3/3     3            3           43m

NAMESPACE   NAME                              DESIRED   CURRENT   READY   AGE
default     replicaset.apps/demo-677cfb9d49   3         3         3       43m
```
### Check component status
@vmahilevskyi ➜ /workspaces/mastering-k8s (lesson-2) $ sudo kubebuilder/bin/kubectl get componentstatuses || true
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
etcd-0               Healthy   ok        
controller-manager   Healthy   ok        
scheduler            Healthy   ok

### Check API server health
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (lesson-2) $ sudo kubebuilder/bin/kubectl get --raw='/readyz?verbose'
[+]ping ok
[+]log ok
[+]etcd ok
[+]etcd-readiness ok
[+]informer-sync ok
[+]poststarthook/start-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/priority-and-fairness-config-consumer ok
[+]poststarthook/priority-and-fairness-filter ok
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