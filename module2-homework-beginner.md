# Objective
- використовуючи гайд, створити, збілдати та запустити свій перший контролер.

# Steps
## Generate deep copy methods
Execution of `controller-gen object paths="./api/..."` fails:
```
controller-gen object paths="./api/..."
-: pattern ./...: directory prefix . does not contain main module or its selected dependencies
Error: not all generators ran successfully
run `controller-gen object paths=./api/... -w` to see all available markers, or `controller-gen object paths=./api/... -h` for usage
```
### !!! Need to intitiate Go module in a first place:
```
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ go mod init github.com/vmahilevskyi/mastering-k8s/my-new-controller
go: creating new go.mod: module github.com/vmahilevskyi/mastering-k8s/my-new-controller
go: to add module requirements and sums:
        go mod tidy
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ go mod tidy
go: finding module for package k8s.io/apimachinery/pkg/runtime
go: finding module for package k8s.io/apimachinery/pkg/apis/meta/v1
go: downloading k8s.io/apimachinery v0.34.1
go: finding module for package k8s.io/apimachinery/pkg/runtime/schema
go: found k8s.io/apimachinery/pkg/apis/meta/v1 in k8s.io/apimachinery v0.34.1
go: found k8s.io/apimachinery/pkg/runtime in k8s.io/apimachinery v0.34.1
go: found k8s.io/apimachinery/pkg/runtime/schema in k8s.io/apimachinery v0.34.1
go: downloading github.com/google/go-cmp v0.7.0
go: downloading github.com/stretchr/testify v1.10.0
go: downloading github.com/go-logr/logr v1.4.2
go: downloading github.com/spf13/pflag v1.0.6
go: downloading golang.org/x/net v0.38.0
go: downloading golang.org/x/text v0.23.0
go: downloading gopkg.in/check.v1 v0.0.0-20161208181325-20d25e280405
```

After Go module is initialize the generation of deep copy methods is succesful
```
controller-gen object paths="./..."
# Check the file is created
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ ls api/v1alpha1 
groupversion.go  newresource_types.go  zz_generated.deepcopy.go
```
## Generate CRDs
```
controller-gen crd:crdVersions=v1 paths=./... output:crd:artifacts:config=config/crd/bases
# Check created file
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ ls -la config/crd/bases 
total 12
drwxrwxrwx+ 2 codespace codespace 4096 Oct 20 19:52 .
drwxrwxrwx+ 3 codespace codespace 4096 Oct 20 19:52 ..
-rw-rw-rw-  1 codespace codespace 1647 Oct 20 19:52 apps.vmah.com_newresources.yaml
```

## Build the controller
```
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ go build -o bin/manager main.go# Check created binary
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ ls -la bin 
total 46860
drwxrwxrwx+ 2 codespace codespace     4096 Oct 20 20:09 .
drwxrwxrwx+ 6 codespace codespace     4096 Oct 20 20:09 ..
-rwxrwxrwx  1 codespace codespace 47970608 Oct 20 20:09 manager
```

## Install CRD
```
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ kubectl apply -f config/crd/bases/apps.vmah.com_newresources.yaml 
customresourcedefinition.apiextensions.k8s.io/newresources.apps.vmah.com created
```

## Run the controller
```
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ ./bin/manager &
[1] 83940
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ 2025-10-20T20:15:27Z  INFO    controller-runtime.metrics Starting metrics server
2025-10-20T20:15:27Z    INFO    controller-runtime.metrics      Serving metrics server  {"bindAddress": ":8080", "secure": false}
2025-10-20T20:15:27Z    INFO    Starting EventSource    {"controller": "newresource", "controllerGroup": "apps.vmah.com", "controllerKind": "NewResource", "source": "kind source: *v1alpha1.NewResource"}
2025-10-20T20:15:27Z    INFO    Starting Controller     {"controller": "newresource", "controllerGroup": "apps.vmah.com", "controllerKind": "NewResource"}
2025-10-20T20:15:27Z    INFO    Starting workers        {"controller": "newresource", "controllerGroup": "apps.vmah.com", "controllerKind": "NewResource", "worker count": 1}
```

## Test the controller
```
# Create resource
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ cat <<EOF | kubectl apply -f -
apiVersion: apps.vmah.com/v1alpha1
kind: NewResource
metadata:
  name: my-very-first-resource
spec:
  foo: "hello world!!"
EOF
newresource.apps.vmah.com/my-very-first-resource created

# Related logs from the controller
2025-10-20T20:17:02Z    INFO    Reconciling     {"controller": "newresource", "controllerGroup": "apps.vmah.com", "controllerKind": "NewResource", "NewResource": {"name":"my-very-first-resource","namespace":"default"}, "namespace": "default", "name": "my-very-first-resource", "reconcileID": "9678555e-39fd-4862-9fc2-f5652e9b7500", "name": "my-very-first-resource", "namespace": "default"}
2025-10-20T20:17:02Z    INFO    Reconciling     {"controller": "newresource", "controllerGroup": "apps.vmah.com", "controllerKind": "NewResource", "NewResource": {"name":"my-very-first-resource","namespace":"default"}, "namespace": "default", "name": "my-very-first-resource", "reconcileID": "9f3b4195-aee8-4325-b443-d506c0d78e4b", "name": "my-very-first-resource", "namespace": "default"}
```

## Verify the controller is working:
```
@vmahilevskyi ➜ /workspaces/mastering-k8s/my-first-controller (module2) $ kubectl get newresource my-very-first-resource -o yaml
apiVersion: apps.vmah.com/v1alpha1
kind: NewResource
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps.vmah.com/v1alpha1","kind":"NewResource","metadata":{"annotations":{},"name":"my-very-first-resource","namespace":"default"},"spec":{"foo":"hello world!!"}}
  creationTimestamp: "2025-10-20T20:17:02Z"
  generation: 1
  name: my-very-first-resource
  namespace: default
  resourceVersion: "2816"
  uid: ae973472-f5b8-4572-a3a4-f663d4df284c
spec:
  foo: hello world!!
status:
  ready: true
```