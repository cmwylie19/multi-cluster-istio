# Gloo Mesh Demo
_Proof of concept for multi-cluster Istio using Gloo Mesh._

## Prereqs
- Create two clusters
- Management plane cluster needs to be >= medium

_Set environmental variables for each cluster_
```
REMOTE_CONTEXT=gke_solo-test-236622_us-east1-b_cw-remote-cluster
MGMT_CONTEXT=gke_solo-test-236622_us-east1-b_cw-mgmt-cluster
```
GLOO_MESH_LICENSE_KEY="$(cat ~/Playground/licensing/mesh.key)"
## Install Gloo Mesh
**Helm installation**
```
helm repo add gloo-mesh-enterprise https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-enterprise


k create ns gloo-mesh
 helm install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise --namespace gloo-mesh --set enterprise-networking.metricsBackend.prometheus.enabled=true --set licenseKey=$GLOO_MESH_LICENSE_KEY
```

## uninstall
```
helm uninstall -n gloo-mesh gloo-mesh-enterprise
```

<!-- ## Update Enterprise Networking
Path the enterprise networking to be of type loadbalancer
```
kubectl -n gloo-mesh patch svc enterprise-networking -p '{"spec": {"type": "LoadBalancer"}}' 
``` -->

## Install Istio in each cluster

Management Cluster
```
cat << EOF | istioctl manifest install -y --context $MGMT_CONTEXT -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: example-istiooperator
  namespace: istio-system
spec:
  profile: minimal
  meshConfig:
    enableAutoMtls: true
    defaultConfig:
      envoyMetricsService:
        address: enterprise-agent.gloo-mesh:9977
      envoyAccessLogService:
        address: enterprise-agent.gloo-mesh:9977
      proxyMetadata:
        # Enable Istio agent to handle DNS requests for known hosts
        # Unknown hosts will automatically be resolved using upstream dns servers in resolv.conf
        ISTIO_META_DNS_CAPTURE: "true"
        GLOO_MESH_CLUSTER_NAME: mgmt-cluster
  components:
    # Istio Gateway feature
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        env:
          - name: ISTIO_META_ROUTER_MODE
            value: "sni-dnat"
        service:
          type: LoadBalancer
          ports:
            - port: 80
              targetPort: 8080
              name: http2
            - port: 443
              targetPort: 8443
              name: https
            - port: 15443
              targetPort: 15443
              name: tls
              nodePort: 32001
  values:
    global:
      multiCluster:
        clusterName: mgmt-cluster
      pilotCertProvider: istiod
EOF
```

Remote Cluster
```
cat << EOF | istioctl manifest install -y --context $REMOTE_CONTEXT -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: example-istiooperator
  namespace: istio-system
spec:
  profile: minimal
  meshConfig:
    enableAutoMtls: true
    defaultConfig:
      envoyMetricsService:
        address: enterprise-agent.gloo-mesh:9977
      envoyAccessLogService:
        address: enterprise-agent.gloo-mesh:9977
      proxyMetadata:
        # Enable Istio agent to handle DNS requests for known hosts
        # Unknown hosts will automatically be resolved using upstream dns servers in resolv.conf
        ISTIO_META_DNS_CAPTURE: "true"
        GLOO_MESH_CLUSTER_NAME: remote-cluster
  components:
    # Istio Gateway feature
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        env:
          - name: ISTIO_META_ROUTER_MODE
            value: "sni-dnat"
        service:
          type: LoadBalancer
          ports:
            - port: 80
              targetPort: 8080
              name: http2
            - port: 443
              targetPort: 8443
              name: https
            - port: 15443
              targetPort: 15443
              name: tls
              nodePort: 32001
  values:
    global:
      multiCluster:
        clusterName: remote-cluster
      pilotCertProvider: istiod
EOF
```


## Register Clusters
```
SVC=$(kubectl --context $MGMT_CONTEXT -n gloo-mesh get svc enterprise-networking -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):9900

meshctl cluster register enterprise   --remote-context=$REMOTE_CONTEXT   --relay-server-address $SVC  remote-cluster 
meshctl cluster register enterprise   --remote-context=$MGMT_CONTEXT   --relay-server-address $SVC  mgmt-cluster
```

## DeRegister Cluster
```
✗ meshctl cluster deregister enterprise remote-cluster
✗ meshctl cluster deregister enterprise mgmt-cluster
```



## Using Gloo Mesh
Get kubernetes clustes  
`k get kubernetesclusters -n gloo-mesh`

Describe the Virtual Mesh  
`meshctl describe mesh`

Deploy Bookinfo Application in Management Cluster:  
```
kubectl --context $MGMT_CONTEXT label namespace default istio-injection=enabled
# deploy bookinfo application components for all versions less than v3
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/platform/kube/bookinfo.yaml -l 'app,version notin (v3)'
# deploy all bookinfo service accounts
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/platform/kube/bookinfo.yaml -l 'account'
# configure ingress gateway to access bookinfo
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/networking/bookinfo-gateway.yaml
````

Deploy Bookinfo Application in Remote Cluster:  
```
kubectl --context $REMOTE_CONTEXT label namespace default istio-injection=enabled
# deploy all bookinfo service accounts and application components for all versions
kubectl --context $REMOTE_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/platform/kube/bookinfo.yaml
# configure ingress gateway to access bookinfo
kubectl --context $REMOTE_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/networking/bookinfo-gateway.yaml
```

## FailOver
Make reviews unavailable to the first cluster:   
```
kubectl --context $MGMT_CONTEXT patch deploy reviews-v1 --patch '{"spec": {"template": {"spec": {"containers": [{"name": "reviews","command": ["sleep", "20h"]}]}}}}'
kubectl --context $MGMT_CONTEXT patch deploy reviews-v2 --patch '{"spec": {"template": {"spec": {"containers": [{"name": "reviews","command": ["sleep", "20h"]}]}}}}'
```

Validate requests are being handled by the second cluster:  
```
kubectl --context cluster2 logs -l app=reviews -c istio-proxy -f
```

Enable reviews traffic in first cluster:
```
kubectl --context $MGMT_CONTEXT patch deployment reviews-v1  --type json   -p '[{"op": "remove", "path": "/spec/template/spec/containers/0/command"}]'
kubectl --context $MGMT_CONTEXT patch deployment reviews-v2  --type json   -p '[{"op": "remove", "path": "/spec/template/spec/containers/0/command"}]'
```
Get workloads in the mesh  
`k -n gloo-mesh get workloads`

Get Destinations in the mesh  
`k -n gloo-mesh get workloads`


Portfoward the Product Page to 8080 Local:  
`k port-forward -n bookinfo svc/productpage 8080:9080`


Create RoleBinding for user
```
apiVersion: rbac.enterprise.mesh.gloo.solo.io/v1
kind: RoleBinding
metadata:
  labels:
    app: gloo-mesh
  name: admin-role-binding2
  namespace: gloo-mesh
spec:
  roleRef:
    name: admin-role
    namespace: gloo-mesh
  subjects:
    - kind: User
      name: casey.wylie@solo.io
```

## AccessLogs 
Update config map  
`kubectl --context $MGMT_CONTEXT -n istio-system get configmap istio -oyaml`  

Ensure that the following highlighted entries exist in the ConfigMap:
```
data:
  mesh:
    defaultConfig:
      envoyAccessLogService:
        address: enterprise-agent.gloo-mesh:9977
      proxyMetadata:
        GLOO_MESH_CLUSTER_NAME: your-gloo-mesh-registered-cluster-name
```
Restart istio injected workloads:   
```
k rollout restart deploy/details-v1 deploy/productpage-v1 deploy/ratings-v1 deploy/reviews-v1 deploy/reviews-v2
```

Remote:   
```
k rollout restart deploy/details-v1 deploy/productpage-v1 deploy/ratings-v1 deploy/reviews-v1 deploy/reviews-v2 deploy/reviews-v3 --context $REMOTE_CONTEXT
```


Create AccessLog Record:
```
apiVersion: observability.enterprise.mesh.gloo.solo.io/v1
kind: AccessLogRecord
metadata:
  name: access-log-all
  namespace: gloo-mesh
spec:
  filters: []
```



Send Requests

Curl Enterprise Networking pod to localhost 8080
```
kubectl --context $MGMT_CONTEXT -n gloo-mesh port-forward deploy/enterprise-networking 8080
```

Curl Logs:  
`curl -XPOST 'localhost:8080/v0/observability/logs?pretty'`


## Metrics
# port-forward enterprise networking  
kubectl -n gloo-mesh port-forward deploy/enterprise-networking 8080  

# request metrics  
curl localhost:8080/metrics  

kubectl -n gloo-mesh port-forward deploy/prometheus-server 9090  


## Upgrade 
Install the CRDS
```
https://raw.githubusercontent.com/solo-io/gloo-mesh/v1.1.0-beta1/install/helm/gloo-mesh-crds/crds/discovery.mesh.gloo.solo.io_v1_crds.yaml
```

Uninstall Gloo
```
helm uninstall -n gloo-mesh gloo-mesh-enterprise
```
Upgrade the helm chart
```
helm upgrade --install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise  --version 1.1.0-beta3 --namespace gloo-mesh --set enterprise-networking.metricsBackend.prometheus.enabled=true --set licenseKey=$GLOO_MESH_LICENSE_KEY   --force
```