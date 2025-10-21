# Objective
- спробувати побудувати gitops кільце для аплікейшену з imageupdate автоматизацією

# Steps
## Fix controlplane.sh
I've had to uncomment `source ./download.sh` and `source ./pki.sh` and put them after `sudo tee /var/lib/kubelet/config.yaml` because `pki.sh` requires it.

## Add missing kube-proxy binary
Fixed by updating `download.sh`:
```
if [ ! -f "kubebuilder/bin/kube-proxy" ]; then
    echo "Downloading kube-proxy..."
    sudo curl -L "https://dl.k8s.io/v1.30.0/bin/linux/amd64/kube-proxy" -o kubebuilder/bin/kube-proxy
    sudo chmod 755 kubebuilder/bin/kube-proxy
fi
```
## Install FluxInstance
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (module3) $ k apply -f gitless/FluxInstance.yaml  
I1021 18:33:23.609365   36670 controller.go:615] quota admission added evaluator for: fluxinstances.fluxcd.controlplane.io
fluxinstance.fluxcd.controlplane.io/flux created
```
## Setup required K8s secrets
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (module3) $ read -s "GITHUB_TOKEN?Enter GitHub token: "
Enter GitHub token: %              
@vmahilevskyi ➜ /workspaces/mastering-k8s (module3) $ k create secret generic github-auth \      
  --from-literal=username=git \
  --from-literal=password=${GITHUB_TOKEN} \
  -n flux-system
secret/github-auth created
@vmahilevskyi ➜ /workspaces/mastering-k8s (module3) $ k create secret docker-registry ghcr-auth \
  --docker-server=ghcr.io \
  --docker-username=vmahilevskyi \
  --docker-password=${GITHUB_TOKEN} \
  -n flux-system
secret/ghcr-auth created
```
## Install CoreDNS
I've noticed that flux-operator didn't launch other controllers and didn't install additional CRDs. I realized it when I got the error message while executing `k apply -f gitless/gitops.yaml`
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (module3) $ k apply -f gitless/gitops.yaml 
resource mapping not found for name: "flux-system" namespace: "flux-system" from "gitless/gitops.yaml": no matches for kind "GitRepository" in version "source.toolkit.fluxcd.io/v1"
ensure CRDs are installed first
```

The following message was in the events of FluxInstance:
```
Warning  ArtifactFailed           38s (x2 over 13m)  flux-operator  fetch failed: pulling artifact oci://ghcr.io/controlpla
neio-fluxcd/flux-operator-manifests failed: Get "https://ghcr.io/v2/": context deadline exceeded
```

I've installed CoreDNS. Initially, I didn't specify `service.clusterIP` parameter and spent some time to figure out why DNS resolution isn't working.
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (module3) $ helm repo add coredns https://coredns.github.io/helm 
helm --namespace=kube-system upgrade --install coredns coredns/coredns --set service.clusterIP=10.10.0.53 
"coredns" has been added to your repositories
I1021 19:25:21.553329   36670 alloc.go:330] "allocated clusterIPs" service="kube-system/coredns" clusterIPs={"IPv4":"10.10.0.224"}
I1021 19:25:21.564265   36670 controller.go:615] quota admission added evaluator for: deployments.apps
I1021 19:25:21.583551   36670 controller.go:615] quota admission added evaluator for: replicasets.apps
NAME: coredns
LAST DEPLOYED: Tue Oct 21 19:25:21 2025
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CoreDNS is now running in the cluster as a cluster-service.

It can be tested with the following:

1. Launch a Pod with DNS tools:

kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools

2. Query the DNS server:

/ # host kubernetes
I1021 19:25:21.639293   36885 topology_manager.go:215] "Topology Admit Handler" podUID="f52caa44-6195-4d7d-a8e8-c8eb71b647dd" podNamespace="kube-system" podName="coredns-6bdd95cdf-647n6"
I1021 19:25:21.691050   36813 replica_set.go:676] "Finished syncing" logger="replicaset-controller" kind="ReplicaSet" key="kube-system/coredns-6bdd95cdf" duration="103.007576ms"
```

And only then flux-operator was able to download and install controllers and CRDs.

## Create all required k8s secrets
```
k create secret docker-registry ghcr-auth \
  --docker-server=ghcr.io \    
  --docker-username=vmahilevskyi \         
  --docker-password=${GITHUB_TOKEN} \
  -n flux-system

k create secret generic github-auth \      
  --from-literal=username=git \
  --from-literal=password=${GITHUB_TOKEN} \
  -n flux-system
```

## Install gitops stack
```
k apply -f gitless/gitops.yaml
```

## Create secret in demo NS
```
k create secret generic github-auth \      
  --from-literal=username=git \
  --from-literal=password=${GITHUB_TOKEN} \
  -n demo
```

## Envoy Gateway
I've intentionally skipped Envoy Gateway installation. 

# Verifications

https://github.com/vmahilevskyi/mastering-k8s-flux-gitops-dev/tree/gitops  
https://github.com/vmahilevskyi/mastering-k8s-kbot-src/tree/gitops

## Helm Chart version change
Initially, application's Helm Chart installation has failed because Envoy Gateway and it's CRDs weren't installed:
```
unable to build kubernetes objects from release manifest: resource m
apping not found for name: "kbot" namespace: "" from "": no matches for kind "HTTPRoute" in versio
n "gateway.networking.k8s.io/v1"
```
Interesting fact, despite we define `GitRepository` as [sourceRef](https://github.com/vmahilevskyi/mastering-k8s-flux-gitops-dev/blob/main/clusters/demo/kbot-hr.yaml#L12) in HelmRelease the [Helm Controller](https://fluxcd.io/flux/components/helm/) didn't detect changes [this](https://github.com/vmahilevskyi/mastering-k8s-kbot-src/commit/9c0dc0cdf12ca993488c263cd509194f2431698b) and [this](https://github.com/vmahilevskyi/mastering-k8s-kbot-src/commit/2a77ed0440d590ab7e4f5e425a4a73b5f9be4fe1) to Helm Chart without updated version.

Once application's Helm Chart version [was updated](https://github.com/vmahilevskyi/mastering-k8s-kbot-src/commit/e5a5ded6a66c621f6da817d7012538f46341c6b5) HelmRelease Flux resource detected the change and Helm install was successful:
```
Events:                                                                                           
  Type     Reason            Age                  From             Message                        
  ----     ------            ----                 ----             -------                        
  Normal   HelmChartCreated  48m                  helm-controller  Created HelmChart/demo/demo-kbo
t with SourceRef 'GitRepository/demo/kbot'                                                        
  Warning  InstallFailed     6m2s (x15 over 48m)  helm-controller  Helm install failed for release
 demo/kbot with chart kbot@2.2.5: unable to build kubernetes objects from release manifest: resour
ce mapping not found for name: "kbot" namespace: "" from "": no matches for kind "HTTPRoute" in ve
rsion "gateway.networking.k8s.io/v1"                                                              
ensure CRDs are installed first                                                                   
  Normal  InstallSucceeded  2m45s  helm-controller  Helm install succeeded for release demo/kbot.v
1 with chart kbot@2.2.6 
``` 
## Application image tag change

[Commit](https://github.com/vmahilevskyi/mastering-k8s-flux-gitops-dev/commit/6e712fc1c5758c6daccbd8ae053a33a8795a9cc8). 
Short SHA - 6e712fc

HelmRelese Flux resouce event:
```
Normal  UpgradeSucceeded  75s  helm-controller  Helm upgrade succeeded for release demo/kbot.v2 
with chart kbot@2.2.6 
```

Deployment kbot event:
```
Normal  ScalingReplicaSet  109s  deployment-controller  Scaled up replica set kbot-5b46676d to 1
  Normal  ScalingReplicaSet  106s  deployment-controller  Scaled down replica set kbot-5968b465c t
o 0 from 1   
```

GitRepository flux-system:
```
Artifact:                                                                                       
    Digest:            sha256:cd76c9600898f607c16854c8f69f83da81c89d0cdf6adf057dfd33bc32979373    
    Last Update Time:  2025-10-21T22:10:35Z                                                       
    Path:              gitrepository/flux-system/flux-system/6e712fc1c5758c6daccbd8ae053a33a8795a9
cc8.tar.gz                                                                                        
    Revision:          gitops@sha1:6e712fc1c5758c6daccbd8ae053a33a8795a9cc8                       
    Size:              591                                                                        
    URL:               http://source-controller.flux-system.svc.cluster.local./gitrepository/flux-
system/flux-system/6e712fc1c5758c6daccbd8ae053a33a8795a9cc8.tar.gz 
```
## Check the application
```
@vmahilevskyi ➜ /workspaces/mastering-k8s (module3) $ curl http://loca
lhost:8080
Welcome to kbot server!
Version: v1.1.0-2a77ed0-amd64
```
Git commit SHA `2a77ed0` corresponds to the release https://github.com/vmahilevskyi/mastering-k8s-kbot-src/releases/tag/v1.1.0
