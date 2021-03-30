# Gloo Mesh Demo
_Proof of concept for multi-cluster Istio using Gloo Mesh._

## Prereqs
- Install Kind
- Install meshctl cli tool

## Install
Spin up two kind clusters:  
`meshctl demo istio-multicluster init`

Set environmental variables for each cluster
```
export MGMT_CONTEXT=kind-mgmt-cluster
export REMOTE_CONTEXT=kind-remote-cluster
```

Get pods from the management cluster:  
`kubectl --context $MGMT_CONTEXT get po -n gloo-mesh`

Check the install:  
`meshctl check`

Install Gloo Mesh enterprise:  
`meshctl install enterprise --license $GLOO_MESH_LICENSE_KEY`

Install Istio in Management Plane:  
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

Install Istio in Control Plane: 
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
              nodePort: 32000
  values:
    global:
      pilotCertProvider: istiod
EOF
```

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