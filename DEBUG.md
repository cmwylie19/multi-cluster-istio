# Debugging

## Cluster Registration
Ensure `client` and `server` have equal values in the `relay-identity-token-secrets`.  

Verify `relay-identity-token-secret` and `relay-root-tls-secret` in the `gloo-mesh` namespace on the client and server clusters have equal values.