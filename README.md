# Gloo Mesh Demo
_Proof of concept for multi-cluster Istio using Gloo Mesh._

## Prereqs
- Create two clusters
- Management plane cluster needs to be >= medium

_Set environmental variables for each cluster_
```
REMOTE_CONTEXT=gke_solo-test-236622_us-east1-b_cw-remote-cluster
MGMT_CONTEXT=gke_solo-test-236622_us-east1-b_cw-management-cluster
```
## Install Gloo Mesh
Helm installation
```
helm repo add gloo-mesh-enterprise https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-enterprise


k create ns gloo-mesh
helm install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise --namespace gloo-mesh  --set licenseKey=$GLOO_MESH_LICENSE_KEY
```

## Update Enterprise Networking
Path the enterprise networking to be of type loadbalancer
```
kubectl -n gloo-mesh patch svc enterprise-networking -p '{"spec": {"type": "LoadBalancer"}}' 
```
Patch the istio ingressgateway to be of type loadbalancer 
```
kubectl -n istio-system patch svc istio-ingressgateway -p '{"spec": {"type": "LoadBalancer"}}' --context $MGMT_CONTEXT
kubectl -n istio-system patch svc istio-ingressgateway -p '{"spec": {"type": "LoadBalancer"}}' --context $REMOTE_CONTEXT
```

## Register Clusters
```
SVC=$(kubectl --context $MGMT_CONTEXT -n gloo-mesh get svc enterprise-networking -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
SVC=$SVC:9900

meshctl cluster register enterprise   --remote-context=$REMOTE_CONTEXT   --relay-server-address $SVC  remote-cluster 
meshctl cluster register enterprise   --remote-context=$MGMT_CONTEXT   --relay-server-address $SVC  mgmt-cluster
```

## DeRegister Cluster
```
✗ meshctl cluster deregister enterprise remote-cluster
```

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
      proxyMetadata:
        # Enable Istio agent to handle DNS requests for known hosts
        # Unknown hosts will automatically be resolved using upstream dns servers in resolv.conf
        ISTIO_META_DNS_CAPTURE: "true"
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
          type: NodePort
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
      pilotCertProvider: istiod
EOF
```
Remote Cluster
From istio 1.8/bin
```
cat << EOF | ./istioctl manifest install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: gloo-mesh-istio
  namespace: istio-system
spec:
  # This value is required for Gloo Mesh Istio
  # This value can be any Gloo Mesh Istio tag
  profile: minimal
  meshConfig:
    enableAutoMtls: true
    defaultConfig:
      proxyMetadata:
        # Enable Istio agent to handle DNS requests for known hosts
        # Unknown hosts will automatically be resolved using upstream dns servers in resolv.conf
        ISTIO_META_DNS_CAPTURE: "true"
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
              nodePort: 32000
  values:
    global:
      pilotCertProvider: istiod
EOF
```

## uninstall
```
helm uninstall -n gloo-mesh gloo-mesh-enterprise
```


Get pods from the management cluster:  
`kubectl --context $MGMT_CONTEXT get po -n gloo-mesh`

Check the install:  
`meshctl check`

Install Gloo Mesh enterprise:  
`meshctl install enterprise --license $GLOO_MESH_LICENSE_KEY`


## Using Gloo Mesh
Get kubernetes clustes  
`k get kubernetesclusters -n gloo-mesh`

Describe the Virtual Mesh  
`meshctl describe mesh`

Deploy Bookinfo Application in Management Cluster:  
`kubectl config use-context $MGMT_CONTEXT`

`kubectl create ns bookinfo`
`kubectl label namespace bookinfo istio-injection=enabled`
​
`kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml -l 'app,version notin (v3)'`
`kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml -l 'account'`

Deploy Bookinfo Application in Remote Cluster:  
`kubectl config use-context $REMOTE_CONTEXT`  

`kubectl create ns bookinfo`  
`kubectl label namespace bookinfo istio-injection=enabled`  
​
`kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml -l 'app,version in (v3)'`   
`kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml -l 'service=reviews'`  
`kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml -l 'account=reviews'`   
`kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml -l 'app=ratings' `  
`kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml -l 'account=ratings'`   


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