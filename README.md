# mcp-gateway
An envoy-based MCP Gateway

# Installation

Deploy Gateway API CRDs (optional)

```
kubectl apply -k "https://github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.3.0"
```

Deploy Envoy Gateway
```
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.4.1 -n envoy-gateway-system --create-namespace
kubectl apply -f envoy/gateway.yaml
kubectl apply -f envoy/envoyproxy.yaml
```

Deploy Envoy Gateway helper
```
kubectl apply -f envoy/gateway-helper.yaml
```

Deploy Envoy WASM filter

```
kubectl apply -f envoy/envoyextensionpolicy.yaml
```

# Test Envoy filter

At this point, you might want to test out if everything is in place by running

```
kubectl -n envoy-gateway-system port-forward service/envoy-mcp-gateway-eg-2141f0d0 8000:80
```

Then start the MCP inspector:

```
DANGEROUSLY_OMIT_AUTH=true npx @modelcontextprotocol/inspector
```

and use Streamable HTTP transport with URL http://localhost:8000/mcp/. You should be able to connect, and
run list tool. However, as nothing is registered yet, no tools will be listed.


# Deploy app

Make the following change before deploying agent and tools

```
kubectl patch configmap environments -n default --type='json' -p='[{"op": "replace", "path": "/data/mcp-weather", "value": "[\n  {\"name\": \"MCP_URL\", \"value\": 
  \"http://envoy-mcp-gateway-eg-2141f0d0.envoy-gateway-system.svc.cluster.local:80/mcp/\"},\n  {\"name\": \"MCP_TRANSPORT\", \"value\": \"streamable-http\"},\n  {\"name\": \"ACP_MCP_TRANSPORT\", \"value\": \"streamable_http\"}\n]"}]'
```

Then proceed to import ACP Weather Agent and MCP Weather Tool. However, if the agent and/or tool is already running, we need to remove them first and rebuild.

```
kubectl delete components acp-weather-service
kubectl delete components weather-tool
```

and then delete the cached images from the worker nodes
```
podman exec -it [vm id] bash
crictl images | grep weather
crictl rmi [above images]
```

After these are cleaned up, go to the Kagenti UI and rebuild the weather agent and tool.

# Register MCP server

Run the following command:
```
kubectl apply -f deploy/weather.yaml
```

Check in the gateway helper pod if it is registered:
```
kubectl -n mcp-gateway logs mcp-gateway-server-6c66ff9bcb-5hgzq
```

