---
Source: https://agentgateway.dev/docs/kubernetes/latest/quickstart/install/
Date: 06/06/2026
---

# Install agentgateway

Install the agentgateway control plane and get agentgateway running in your cluster. Agentgateway is an open source, AI-first data plane that provides connectivity for agents, MCP tools, LLMs, and inferences in any environment. In Kubernetes environments, you can use agentgateway as the control plane to quickly spin up and manage the lifecycle of agentgateway proxies. The control plane translates Kubernetes Gateway API and agentgateway’s Kubernetes custom resources such as AgentgatewayPolicy and AgentgatewayBackend into proxy configuration for the data plane

## Kind cluster creation

```sh
kind create cluster
```

## Install

The following steps get you started with a basic installation. 

Deploy the Kubernetes Gateway API CRDs:
```sh
kubectl apply --server-side --force-conflicts -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
```

Deploy the CRDs for the agentgateway control plane by using Helm:
```sh
helm upgrade -i agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds \
--create-namespace --namespace agentgateway-system \
--version v1.2.1 \
--set controller.image.pullPolicy=Always
```

Install the agentgateway control plane by using Helm:
```sh
helm upgrade -i agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
  --namespace agentgateway-system \
  --version v1.2.1 \
  --set controller.image.pullPolicy=Always \
  --set controller.extraEnv.KGW_ENABLE_GATEWAY_API_EXPERIMENTAL_FEATURES=true \
  --wait
```

Make sure that the agentgateway control plane is running:
```sh
kubectl get pods -n agentgateway-system
```

## Set up an agentgateway proxy 

Create a Gateway that uses the agentgateway GatewayClass. The following example sets up a Gateway that uses the default agentgateway proxy template:
```sh
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
  - protocol: HTTP
    port: 80
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

Verify that the agentgateway proxy is created:
- Gateway: Note that it might take a few minutes for an address to be assigned.
- Pod for agentgateway-proxy: The pod has one container: agent-gateway.

```sh
kubectl get gateway agentgateway-proxy -n agentgateway-system
kubectl get deployment agentgateway-proxy -n agentgateway-system
```

Verify that the external IP has been created and is not pending:
```sh
kubectl get svc -n agentgateway-system agentgateway-proxy
```

(locally should remain pending)

Port-forward for local testing:
```sh
kubectl port-forward deployment/agentgateway-proxy -n agentgateway-system 8080:80
```

## Cleanup

Cleanup:
```sh
helm uninstall agentgateway agentgateway-crds -n agentgateway-system
```
