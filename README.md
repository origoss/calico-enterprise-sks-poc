- [Introduction](#orgfbaabef)
- [Provisioning the infrastructure](#org8e05233)
  - [Provisioning SKS](#org26254b0)
  - [Preparing persistent storage](#orgae7a3a3)
- [Deploying Calico Enterprise](#org76eff05)
  - [Deploying the operators](#org3ab25e2)
  - [Deploying the license and pull secrets](#orgdb6b745)
  - [Deploying Tigera custom resources](#org12cb684)
  - [Provisioning worker nodes](#orgd3b1925)
  - [Deploying license key](#org5c2d243)
  - [Status after the deployment](#org16083f0)
- [Troubleshooting](#orgc479435)
- [Workaround](#orga7ef7c4)



<a id="orgfbaabef"></a>

# Introduction

This document shows how to deploy Calico Enterprise on top of Exoscale SKS.


<a id="org8e05233"></a>

# Provisioning the infrastructure


<a id="org26254b0"></a>

## Provisioning SKS

Currently, we don't have any SKS clusters:

```bash
exo compute sks list
```

![img](exo-compute-sks-list.gif)

Let's create one with the name `ceosks-poc` without any CNI and worker nodes:

```bash
exo compute sks create ceosks-poc \
    --no-cni                      \
    --nodepool-size 0             \
    --zone at-vie-1
```

![img](exo-compute-sks-create.gif)

Now, we can obtain the kubeconfig file:

```bash
exo compute sks kubeconfig ceosks-poc admin \
    --zone at-vie-1                         \
    -g system:masters                       \
    -t $((86400 * 7)) > "$KUBECONFIG"
chmod 0600 "$KUBECONFIG"
```

![img](obtain-kubeconfig.gif)


<a id="orgae7a3a3"></a>

## Preparing persistent storage

Persistent storage will be provided by Longhorn. Let's deploy it with `helm`.

```bash
helm install my-longhorn longhorn      \
     --version 1.7.0                   \
     --repo https://charts.longhorn.io \
     --namespace longhorn              \
     --create-namespace
```

![img](deploy-longhorn.gif)

Let's create a StorageClass called `tigera-elasticsearch`.

The content of the `storageclass-tigera-elasticsearch.yaml` file is:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tigera-elasticsearch
provisioner: driver.longhorn.io
volumeBindingMode: Immediate
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fromBackup: ""
  fsType: "ext4"
  dataLocality: "disabled"
```

Let's deploy it:

```bash
kubectl apply -f storageclass-tigera-elasticsearch.yaml
```

![img](deploy-storageclass.gif)


<a id="org76eff05"></a>

# Deploying Calico Enterprise


<a id="org3ab25e2"></a>

## Deploying the operators

Let's deploy the Tigera operator:

```bash
kubectl create -f https://downloads.tigera.io/ee/v3.19.2/manifests/tigera-operator.yaml
```

![img](deploy-tigera-operator.gif)

Then, we can deploy the Prometheus Operator:

```bash
kubectl create -f https://downloads.tigera.io/ee/v3.19.2/manifests/tigera-prometheus-operator.yaml
```

![img](deploy-prometheus-operator.gif)


<a id="orgdb6b745"></a>

## Deploying the license and pull secrets

Let's deploy the pull secret in the `tigera-operator` namespace.

```bash
kubectl create secret generic tigera-pull-secret           \
  --type=kubernetes.io/dockerconfigjson -n tigera-operator \
  --from-file=.dockerconfigjson=tigera-partners-origoss-auth.json
```

![img](deploy-pull-secret-tigera.gif)

Now, let's overwrite the pull secret in the `tigera-prometheus` namespace.

```bash
kubectl delete secret tigera-pull-secret -n tigera-prometheus
kubectl create secret generic tigera-pull-secret             \
  --type=kubernetes.io/dockerconfigjson -n tigera-prometheus \
  --from-file=.dockerconfigjson=tigera-partners-origoss-auth.json
```

![img](deploy-pull-secret-prometheus.gif)


<a id="org12cb684"></a>

## Deploying Tigera custom resources

The next step is to deploy the custom resources:

```bash
kubectl create -f https://downloads.tigera.io/ee/v3.19.2/manifests/custom-resources.yaml
```

![img](deploy-custom-resources.gif)


<a id="orgd3b1925"></a>

## Provisioning worker nodes

At this point, we shall spin up the worker nodes:

```bash
exo compute sks nodepool add                             \
    --zone at-vie-1 ceosks-poc ceosks-poc-worker         \
    --size=2                                             \
    --instance-type c6f99499-7f59-4138-9427-a09db13af2bc \
    --security-group ceosks-poc
```

![img](exo-compute-sks-nodepool-add.gif)

After some time, let's check the status of the Calico components:

```bash
kubectl get tigerastatus
```

![img](kubectl-get-tigerastatus-1.gif)

```
NAME                          AVAILABLE   PROGRESSING   DEGRADED   SINCE
apiserver                     True        False         False      10m
calico                        True        False         False      11m
compliance                                              True
intrusion-detection                                     True
ippools                       True        False         False      31m
log-collector                                           True
log-storage                   True        False         False      31m
log-storage-access                                      True
log-storage-elastic                                     True
log-storage-esmetrics                                   True
log-storage-kubecontrollers                             True
log-storage-secrets           True        False         False      31m
manager                                                 True
monitor                       True        False         False      30m
policy-recommendation                                   True
tiers                                                   True
```


<a id="org5c2d243"></a>

## Deploying license key

Deploying the license key should help with the degraded services. Let's deploy the license key then!

```bash
kubectl create -f license.yml
```

![img](deploy-license-key.gif)


<a id="org16083f0"></a>

## Status after the deployment

After performing the above steps the Calico Enterprise is **not** fully functional.

```bash
kubectl get tigerastatus
```

```
NAME                          AVAILABLE   PROGRESSING   DEGRADED   SINCE
apiserver                     True        False         False      8m23s
calico                        True        False         False      9m28s
compliance                    True        False         False      3m53s
intrusion-detection                                     True
ippools                       True        False         False      10m
log-collector                 True        False         False      3m28s
log-storage                   True        False         False      10m
log-storage-access                                      True
log-storage-dashboards                                  True
log-storage-elastic           False       False         True       4m28s
log-storage-esmetrics                                   True
log-storage-kubecontrollers                             True
log-storage-secrets           True        False         False      10m
manager                                                 True
monitor                       True        False         False      8m58s
policy-recommendation         True        False         False      4m23s
tiers                         True        False         False      4m28s
```

There are some degraded components:

-   intrusion-detection
-   log-storage-access
-   log-storage-dashboards
-   log-storage-elastic
-   log-storage-esmetrics
-   log-storage-kubecontrollers
-   manager


<a id="orgc479435"></a>

# Troubleshooting

Let's investigate what went wrong during the deployment.

There are some suspicious log messages:

```bash
kubectl logs -p --tail=10 -n tigera-eck-operator elastic-operator-0
```

    {"log.level":"info","@timestamp":"2024-09-09T15:04:10.956Z","log.logger":"manager","message":"maxprocs: Updating GOMAXPROCS=1: determined from CPU quota","service.version":"0.0.0-SNAPSHOT+00000000","service.type":"eck","ecs.version":"1.4.0"}
    {"log.level":"info","@timestamp":"2024-09-09T15:04:10.956Z","log.logger":"manager","message":"Setting default container registry","service.version":"0.0.0-SNAPSHOT+00000000","service.type":"eck","ecs.version":"1.4.0","container_registry":""}
    {"log.level":"info","@timestamp":"2024-09-09T15:04:10.956Z","log.logger":"manager","message":"Setting up scheme","service.version":"0.0.0-SNAPSHOT+00000000","service.type":"eck","ecs.version":"1.4.0"}
    {"log.level":"info","@timestamp":"2024-09-09T15:04:10.957Z","log.logger":"manager","message":"Operator configured to manage multiple namespaces","service.version":"0.0.0-SNAPSHOT+00000000","service.type":"eck","ecs.version":"1.4.0","namespaces":["tigera-elasticsearch","tigera-kibana"],"operator_namespace":"tigera-eck-operator"}
    {"log.level":"error","@timestamp":"2024-09-09T15:04:40.958Z","log.logger":"manager.eck-operator","message":"Failed to get API Group-Resources","service.version":"0.0.0-SNAPSHOT+00000000","service.type":"eck","ecs.version":"1.4.0","error":"Get \"https://10.96.0.1:443/api?timeout=1m0s\": dial tcp 10.96.0.1:443: i/o timeout","error.stack_trace":"sigs.k8s.io/controller-runtime/pkg/cluster.New\n\t/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.13.1/pkg/cluster/cluster.go:160\nsigs.k8s.io/controller-runtime/pkg/manager.New\n\t/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.13.1/pkg/manager/manager.go:344\ngithub.com/elastic/cloud-on-k8s/v2/cmd/manager.startOperator\n\t/go/src/github.com/projectcalico/calico/third_party/eck-operator/cloud-on-k8s/cmd/manager/main.go:562\ngithub.com/elastic/cloud-on-k8s/v2/cmd/manager.doRun.func2\n\t/go/src/github.com/projectcalico/calico/third_party/eck-operator/cloud-on-k8s/cmd/manager/main.go:382"}
    {"log.level":"error","@timestamp":"2024-09-09T15:04:40.965Z","log.logger":"manager","message":"Failed to create controller manager","service.version":"0.0.0-SNAPSHOT+00000000","service.type":"eck","ecs.version":"1.4.0","error":"Get \"https://10.96.0.1:443/api?timeout=1m0s\": dial tcp 10.96.0.1:443: i/o timeout","error.stack_trace":"github.com/elastic/cloud-on-k8s/v2/cmd/manager.startOperator\n\t/go/src/github.com/projectcalico/calico/third_party/eck-operator/cloud-on-k8s/cmd/manager/main.go:564\ngithub.com/elastic/cloud-on-k8s/v2/cmd/manager.doRun.func2\n\t/go/src/github.com/projectcalico/calico/third_party/eck-operator/cloud-on-k8s/cmd/manager/main.go:382"}
    {"log.level":"error","@timestamp":"2024-09-09T15:04:40.965Z","log.logger":"manager","message":"Operator stopped with error","service.version":"0.0.0-SNAPSHOT+00000000","service.type":"eck","ecs.version":"1.4.0","error":"Get \"https://10.96.0.1:443/api?timeout=1m0s\": dial tcp 10.96.0.1:443: i/o timeout","error.stack_trace":"github.com/elastic/cloud-on-k8s/v2/cmd/manager.doRun.func2\n\t/go/src/github.com/projectcalico/calico/third_party/eck-operator/cloud-on-k8s/cmd/manager/main.go:384"}
    {"log.level":"error","@timestamp":"2024-09-09T15:04:40.965Z","log.logger":"manager","message":"Shutting down due to error","service.version":"0.0.0-SNAPSHOT+00000000","service.type":"eck","ecs.version":"1.4.0","error":"Get \"https://10.96.0.1:443/api?timeout=1m0s\": dial tcp 10.96.0.1:443: i/o timeout","error.stack_trace":"github.com/elastic/cloud-on-k8s/v2/cmd/manager.doRun\n\t/go/src/github.com/projectcalico/calico/third_party/eck-operator/cloud-on-k8s/cmd/manager/main.go:393\ngithub.com/spf13/cobra.(*Command).execute\n\t/go/pkg/mod/github.com/spf13/cobra@v1.6.1/command.go:916\ngithub.com/spf13/cobra.(*Command).ExecuteC\n\t/go/pkg/mod/github.com/spf13/cobra@v1.6.1/command.go:1044\ngithub.com/spf13/cobra.(*Command).Execute\n\t/go/pkg/mod/github.com/spf13/cobra@v1.6.1/command.go:968\nmain.main\n\t/go/src/github.com/projectcalico/calico/third_party/eck-operator/cloud-on-k8s/cmd/main.go:31\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:271"}
    Error: Get "https://10.96.0.1:443/api?timeout=1m0s": dial tcp 10.96.0.1:443: i/o timeout

The container of the `elastic-operator-0` pod times out when it wants to access the `10.96.0.1:443` TCP port.

This IP address belongs to the `kubernetes.default` service.

```bash
kubectl describe svc/kubernetes
```

```
Name:                     kubernetes
Namespace:                default
Labels:                   component=apiserver
                          provider=kubernetes
Annotations:              <none>
Selector:                 <none>
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.0.1
IPs:                      10.96.0.1
Port:                     https  443/TCP
TargetPort:               30925/TCP
Endpoints:                194.182.185.29:30925
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

The single Endpoint of the Service is `194.182.185.29:30925`. The IP address and the port number can be different for each SKS cluster.

There is a Calico NetworkPolicy deployed in the `tigera-eck-operator` namespace:

```bash
kubectl get networkpolicies.crd.projectcalico.org -n tigera-eck-operator
```

It has the following rules:

```bash
kubectl describe networkpolicies.crd.projectcalico.org \
        allow-tigera.elastic-operator-access -n tigera-eck-operator
```

```
Name:         allow-tigera.elastic-operator-access
Namespace:    tigera-eck-operator
Labels:       projectcalico.org/tier=allow-tigera
Annotations:  projectcalico.org/metadata:
                {"creationTimestamp":"2024-09-09T14:58:35Z","labels":{"projectcalico.org/tier":"allow-tigera"},"ownerReferences":[{"apiVersion":"operator....
API Version:  crd.projectcalico.org/v1
Kind:         NetworkPolicy
Metadata:
  Creation Timestamp:  2024-09-09T14:58:35Z
  Generation:          1
  Resource Version:    144642852
  UID:                 b11b4910-44ab-43e1-9030-8355f30da718
Spec:
  Egress:
    Action:  Allow
    Destination:
      Namespace Selector:  projectcalico.org/name == 'kube-system'
      Ports:
        53
      Selector:  k8s-app == 'kube-dns'
    Protocol:    UDP
    Source:
    Action:  Allow
    Destination:
      Namespace Selector:  projectcalico.org/name == 'default'
      Ports:
      443
        6443
        12388
      Selector:  (provider == 'kubernetes' && component == 'apiserver' && endpoints.projectcalico.org/serviceName == 'kubernetes')
    Protocol:    TCP
    Source:
    Action:  Allow
    Destination:
      Namespace Selector:  projectcalico.org/name == 'tigera-elasticsearch'
      Ports:
        9200
      Selector:  elasticsearch.k8s.elastic.co/cluster-name == 'tigera-secure'
    Protocol:    TCP
    Source:
  Order:     1
  Selector:  k8s-app == 'elastic-operator'
  Tier:      allow-tigera
  Types:
    Egress
Events:  <none>
```

The endpoints of the `kubernetes.default` service can be reached at ports 443, 6443 and 12388. In our case, the endpoint is accepting connections at port **30925**.

We can see similar log messages in another container too:

```bash
kubectl logs -p --tail=10 -n calico-system \
        -l k8s-app=es-calico-kube-controllers
```

    2024-09-09 15:27:07.934 [INFO][1] cmdwrapper.go 56: Starting /usr/bin/kube-controllers
    2024-09-09 15:27:07.997 [INFO][13] main.go 175: Loaded configuration from environment config=&config.Config{LogLevel:"info", WorkloadEndpointWorkers:1, ProfileWorkers:1, PolicyWorkers:1, ServiceWorkers:1, NodeWorkers:1, FederatedServicesWorkers:1, AuthorizationWorkers:1, ManagedClusterWorkers:1, ManagedClusterElasticsearchConfigurationWorkers:1, ManagedClusterLicenseConfigurationWorkers:1, Kubeconfig:"", DoNotInitializeCalico:false, DatastoreType:"kubernetes", DebugUseShortPollIntervals:false, MultiClusterForwardingEndpoint:"https://tigera-manager.tigera-manager.svc:9443", MultiClusterForwardingCA:"/etc/pki/tls/certs/tigera-ca-bundle.crt", OIDCAuthUsernamePrefix:"", OIDCAuthGroupPrefix:"", EnableElasticsearchOIDCWorkaround:true, ElasticUsername:"tigera-ee-kube-controllers", ElasticPassword:"DlBZXk32BlIsJTAw", ElasticHost:"tigera-secure-es-gateway-http.tigera-elasticsearch.svc", ElasticPort:"9200", ElasticCA:"/etc/pki/tls/certs/tigera-ca-bundle.crt", DisableKubeControllersConfigAPI:false, KubeControllersConfigName:"elasticsearch", TenantNamespace:"", UsageReportsPerDay:4, UsageReportRetentionPeriod:"8760h"}
    2024-09-09 15:27:07.998 [INFO][13] main.go 202: Ensuring Calico datastore is initialized
    2024-09-09 15:27:37.999 [ERROR][13] client.go 415: Error getting cluster information config ClusterInformation="default" error=Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0.1:443: i/o timeout
    2024-09-09 15:27:37.999 [INFO][13] client.go 349: Unable to initialize ClusterInformation error=Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0.1:443: i/o timeout
    2024-09-09 15:28:08.027 [INFO][13] client.go 354: Unable to initialize default Tier error=Post "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/tiers": context deadline exceeded
    2024-09-09 15:28:08.027 [INFO][13] main.go 209: Failed to initialize datastore error=Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0.1:443: i/o timeout
    2024-09-09 15:28:08.029 [FATAL][13] main.go 222: Failed to initialize Calico datastore


<a id="orga7ef7c4"></a>

# Workaround

We can allow the port **30925** by deploying a permissive Calico NetworkPolicy with a higher priority into the `tigera-eck-operator` namespace.

Let's consider the following simple NetworkPolicy resource:

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-tigera.tem-bypass
spec:
  tier: allow-tigera
  order: 0
  selector: ''
  serviceAccountSelector: ''
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        ports:
          - '30925'
  types:
    - Egress
```

Let's deploy it:

```bash
kubectl apply -f permissive-networkpolicy.yaml -n tigera-eck-operator
```

    networkpolicy.projectcalico.org/allow-tigera.tem-bypass created

After a while, let's check again the TigeraStatus resources:

```bash
kubectl get tigerastatus
```

```
NAME                          AVAILABLE   PROGRESSING   DEGRADED   SINCE
apiserver                     True        False         False      36m
calico                        True        False         False      37m
compliance                    True        False         False      3m44s
intrusion-detection           True        False         False      6m29s
ippools                       True        False         False      38m
log-collector                 True        False         False      31m
log-storage                   True        False         False      38m
log-storage-access                                      True
log-storage-dashboards                                  True
log-storage-elastic           False       False         True       32m
log-storage-esmetrics                                   True
log-storage-kubecontrollers   True        False         False      6m19s
log-storage-secrets           True        False         False      38m
manager                                                 True
monitor                       True        False         False      37m
policy-recommendation         True        False         False      32m
tiers                         True        False         False      32m
```

It's not perfect yet.

Let's deploy the same permissive NetworkPolicy into the `calico-system` namespace.

```bash
kubectl apply -f permissive-networkpolicy.yaml -n calico-system
```

    networkpolicy.projectcalico.org/allow-tigera.tem-bypass created

Let's wait a couple of minutes then TigerStatuses look like this:

```bash
kubectl get tigerastatus
```

```
NAME                          AVAILABLE   PROGRESSING   DEGRADED   SINCE
apiserver                     True        False         False      43m
calico                        True        False         False      44m
compliance                    True        False         False      72s
intrusion-detection           True        False         False      12m
ippools                       True        False         False      45m
log-collector                 True        False         False      38m
log-storage                   True        False         False      45m
log-storage-access            True        False         False      117s
log-storage-dashboards        True        False         False      2m27s
log-storage-elastic           True        False         False      2m32s
log-storage-esmetrics         True        False         False      2m17s
log-storage-kubecontrollers   True        False         False      12m
log-storage-secrets           True        False         False      45m
manager                       True        False         False      2m7s
monitor                       True        False         False      43m
policy-recommendation         True        False         False      39m
tiers                         True        False         False      39m
```

The workaround helped.
