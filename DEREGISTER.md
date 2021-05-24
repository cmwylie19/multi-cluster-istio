## Deregister cluster
```
kubectl delete accesspolicy -n gloo-mesh --all
kubectl delete trafficpolicy -n gloo-mesh --all
kubectl delete virtualmesh -n gloo-mesh --all
sleep 5;
echo "Deregistering Cluster 1"
meshctl cluster deregister enterprise cluster1
sleep 5;
echo "Deregistering Cluster 2"
meshctl cluster deregister enterprise cluster2
sleep 5;
echo "Deleting secrets in gloo-mesh namespace in Cluster1"
kubectl delete secret --context ${CTX_CLUSTER1} -n gloo-mesh --all
sleep 5;
echo "Deleting secrets in gloo-mesh namespace in Cluster2"
kubectl delete secret --context ${CTX_CLUSTER2} -n gloo-mesh --all
sleep 5;
echo "Registering Cluster 1"
meshctl cluster register --mgmt-context=${MGM_CLUSTER} --remote-context=${CTX_CLUSTER1} --relay-server-address=$SVC:9900 enterprise cluster1 --cluster-domain cluster.local
sleep 5;
echo "Registering Cluster 2"
meshctl cluster register --mgmt-context=${MGM_CLUSTER} --remote-context=${CTX_CLUSTER2} --relay-server-address=$SVC:9900 enterprise cluster1 --cluster-domain cluster.local
```