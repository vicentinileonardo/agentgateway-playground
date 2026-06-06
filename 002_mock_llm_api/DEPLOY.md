---
Source: https://agentgateway.dev/docs/kubernetes/latest/llm/providers/httpbun/
Date: 06/06/2026
---

# Mock LLM with httpbun

Install httpbun for a mock LLM API endpoint that is compatible with the OpenAI API for chat completions.

## Why httpbun?

httpbun is the LLM-testing equivalent of httpbin. Testing AI gateway policies against a real LLM costs money, requires API keys, and introduces network latency. Instead, you can use httpbun as a drop-in OpenAI-compatible LLM backend in Kubernetes, then route traffic to it through agentgateway.

After you deploy httpbun, you can send standard POST /v1/chat/completions requests through agentgateway (including streaming), apply AgentgatewayPolicy resources for rate limiting and auth, and build out the rest of your AI gateway configuration without touching a real LLM.

The /llm endpoint implements the OpenAI chat completions API in the same request format, same response structure, and same streaming SSE protocol. You can customize the mock response content via an httpbun field in the request body, making it useful for testing both the happy path and error cases.

## Prerequisites

- Agentgateway already installed ([Basic installation](../001_basic_installation/INSTALL.md))

## Deploy httpbun in Kubernetes 

```sh
kubectl apply -f- <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbun
  namespace: default
  labels:
    app: httpbun
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbun
  template:
    metadata:
      labels:
        app: httpbun
    spec:
      containers:
        - name: httpbun
          image: sharat87/httpbun
          env:
            - name: HTTPBUN_BIND
              value: "0.0.0.0:3090"
          ports:
            - containerPort: 3090
---
apiVersion: v1
kind: Service
metadata:
  name: httpbun
  namespace: default
  labels:
    app: httpbun
spec:
  selector:
    app: httpbun
  ports:
    - protocol: TCP
      port: 3090
      targetPort: 3090
  type: ClusterIP
EOF
```

Verify the pod is running:

```sh
kubectl get pods -l app=httpbun
```

## Create the backend

Create an AgentgatewayBackend to configure httpbun as an LLM provider. You set the openai provider type, because httpbun implements the OpenAI-compatible API. Then, override the host, port, and path to point at httpbun’s /llm/chat/completions endpoint.

```sh
kubectl apply -f- <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: httpbun-llm
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai:
        model: gpt-4
      host: httpbun.default.svc.cluster.local
      port: 3090
      path: "/llm/chat/completions"
EOF
```

Review the following table to understand this configuration.

|Field |	Value |	Description
| - | - | - 
ai.provider.host |	httpbun.default.svc.cluster.local|	Points to the in-cluster httpbun Service. Because the host is an in-cluster DNS name (not a public HTTPS endpoint), no TLS configuration is required.
ai.provider.port |	3090 |	Matches the httpbun container port.
ai.provider.path |	/llm/chat/completions| 	httpbun’s LLM endpoint (not the default /v1/chat/completions).

## Create the HTTPRoute

Route incoming traffic from the gateway to the AgentgatewayBackend. The route exposes the standard OpenAI-compatible path /v1/chat/completions, so any OpenAI SDK or client can point at the gateway without modification.

Create the HTTPRoute:
```sh
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbun-llm
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: agentgateway-proxy
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1/chat/completions
      backendRefs:
        - name: httpbun-llm
          namespace: agentgateway-system
          group: agentgateway.dev
          kind: AgentgatewayBackend
EOF
```

Verify the route was accepted. Look for `Accepted: True` and `ResolvedRefs: True` in the status. 
If `ResolvedRefs` is `False`, double-check that the AgentgatewayBackend name and namespace in the route match exactly what you created.

```sh
kubectl describe httproute httpbun-llm -n agentgateway-system
```

## Verify the connection

Get the external address of the gateway and save it in an environment variable.

Cloud Provider LoadBalancer:
```sh
export INGRESS_GW_ADDRESS=$(kubectl get svc -n agentgateway-system agentgateway-proxy \
  -o=jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

Port-forward for local testing:

Port-forward the gateway proxy http pod on port 8080.
```sh
kubectl port-forward deployment/agentgateway-proxy -n agentgateway-system 8080:80
```

### Non-streaming request 

Send a standard chat completion request.

```sh
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {"role": "user", "content": "Explain agentgateway in one sentence."}
    ]
  }' | jq
```

Example output:
```json
{
  "model": "gpt-4",
  "usage": {
    "prompt_tokens": 11,
    "completion_tokens": 29,
    "total_tokens": 40
  },
  "choices": [
    {
      "message": {
        "content": "This is a mock chat response from httpbun. I received your messages and I'm responding with this placeholder text.",
        "role": "assistant"
      },
      "finish_reason": "stop",
      "index": 0
    }
  ],
  "created": 1780754181,
  "id": "chatcmpl-47ffc88b48cad43ea737cdb5",
  "object": "chat.completion"
}
```

### Streaming request

```sh
curl -N http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {"role": "user", "content": "Count to three."}
    ],
    "stream": true
  }'
```

Notice the stream of data: chunks in server-sent event format, followed by `data: [DONE]`. 
This output matches the format of an OpenAI streaming response.

```sh
...

data: {"choices":[{"delta":{"content":"text."},"finish_reason":null,"index":0}],"created":1771885787,"id":"chatcmpl-e2e54892bd932edb9549b442","model":"gpt-4","object":"chat.completion.chunk"}

data: {"choices":[{"delta":{},"finish_reason":"stop","index":0}],"created":1771885787,"id":"chatcmpl-e2e54892bd932edb9549b442","model":"gpt-4","object":"chat.completion.chunk"}

data: [DONE]
```

### Custom response content 

Control exactly what the mock LLM returns by including the httpbun field in the request body. 
This is useful for writing deterministic integration tests against policies. You control the response, so you can verify that your gateway transforms, rate limits, or rejects it correctly.

```sh
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello!"}],
    "httpbun": {"content": "Gateway is working perfectly."}
  }' | jq '.choices[0].message.content'
```

Example output:

```sh
"Gateway is working perfectly."
```

## Cleanup

You can remove the resources that you created in this guide.

```sh
kubectl delete httproute httpbun-llm -n agentgateway-system
kubectl delete AgentgatewayBackend httpbun-llm -n agentgateway-system
kubectl delete deployment httpbun -n default
kubectl delete service httpbun -n default
```