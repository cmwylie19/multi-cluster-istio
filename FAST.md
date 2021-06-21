## Fast Install
```
GLOO_MESH_LICENSE_KEY="eyJleHAiOjE2MjY4MjU2MDAsImlhdCI6MTYyNDIzMzYwMCwiayI6IjFPVmVqZyIsImx0IjoidHJpYWwiLCJwcm9kdWN0IjoiZ2xvby1tZXNoIn0.m-mgK5G7ppT4YNuap3nYPtgJfQQk9bma8Gynh46loHQ"
REMOTE_CONTEXT=gke_solo-test-236622_us-east1-b_remote-cluster-cw
MGMT_CONTEXT=gke_solo-test-236622_us-east1-b_mgmt-cluster-cw


helm repo add gloo-mesh-enterprise https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-enterprise

kubectl create ns gloo-mesh

helm install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise --version=1.1.0-beta11 --namespace gloo-mesh --set enterprise-networking.metricsBackend.prometheus.enabled=true --set licenseKey=$GLOO_MESH_LICENSE_KEY

sleep 15;
say "installed gloo mesh enterprise"
echo "\n\n\n"

kubectl --context $MGMT_CONTEXT create ns istio-system
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

sleep 15;
say "Installed Istio in management"
echo "\n\n\n"

kubectl --context $REMOTE_CONTEXT create ns istio-system
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

sleep 15;
say "Installed istio in remote"
echo "\n\n\n"

SVC=$(kubectl --context $MGMT_CONTEXT -n gloo-mesh get svc enterprise-networking -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):9900

meshctl cluster register enterprise   --remote-context=$REMOTE_CONTEXT   --relay-server-address $SVC  remote-cluster 
meshctl cluster register enterprise   --remote-context=$MGMT_CONTEXT   --relay-server-address $SVC  mgmt-cluster

sleep 15; 
say "registered clusters"
echo "\n\n\n"

kubectl --context $MGMT_CONTEXT label namespace default istio-injection=enabled
# deploy bookinfo application components for all versions less than v3
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/platform/kube/bookinfo.yaml -l 'app,version notin (v3)'
# deploy all bookinfo service accounts
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/platform/kube/bookinfo.yaml -l 'account'
# configure ingress gateway to access bookinfo
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/networking/bookinfo-gateway.yaml

sleep 15;
say "Installed bookinfo in management"
echo "\n\n\n"

kubectl --context $REMOTE_CONTEXT label namespace default istio-injection=enabled
# deploy all bookinfo service accounts and application components for all versions
kubectl --context $REMOTE_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/platform/kube/bookinfo.yaml
# configure ingress gateway to access bookinfo
kubectl --context $REMOTE_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/networking/bookinfo-gateway.yaml

say "Installed bookinfo in remote"
```

## Fast Cleanup
```
k delete ns istio-system gloo-mesh;
k delete ns istio-system gloo-mesh --context $REMOTE_CONTEXT;
k delete svc,sa,deploy --all;
k delete svc,sa,deploy --all --context $REMOTE_CONTEXT;
k delete gateway.networking.istio.io/bookinfo-gateway
k delete gateway.networking.istio.io/bookinfo-gateway --context $REMOTE_CONTEXT
k delete vs --all; 
k delete vs --all --context $REMOTE_CONTEXT
say "deleted objects"
```