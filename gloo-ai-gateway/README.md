# AI Gateway

<aside>

💡 Make sure you’re using a non-ARM cluster. AI gateway extension isn’t built for ARM.

</aside>


# Build Docker image

Build and push the docker image for the app

```bash
docker buildx build . --platform linux/amd64,linux/arm64 -t btjimerson/openai-demo:0.1.0

docker push btjimerson/openai-demo:0.1.0
```

# Gloo Gateway

Set the following environment variables:

```bash
export GLOO_GATEWAY_LICENSE_KEY=<gloo-gateway-license-key>

export OPENAI_API_KEY=<openai-api-key>
```

Install the Kubernetes Gateway CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

Install Gloo Gateway with Helm:

```bash
helm install -n gloo-system gloo gloo-ee \
--repo=https://storage.googleapis.com/gloo-ee-helm \
--create-namespace \
--version 1.18.3 \
--set-string license_key=$GLOO_GATEWAY_LICENSE_KEY \
-f- <<EOF
gloo:
  discovery:
    enabled: false
  gatewayProxies:
    gatewayProxy:
      disabled: true
  kubeGateway:
    enabled: true
  gloo:
    disableLeaderElection: true
gloo-fed:
  enabled: false
  glooFedApiserver:
    enable: false
grafana:
  defaultInstallationEnabled: false
observability:
  enabled: false
prometheus:
  enabled: false
EOF

kubectl rollout status -n gloo-system deployments/gloo
```

# HTTP Gateway

Create a gateway with an HTTP listener:

```bash
kubectl apply -f- <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: http
  namespace: gloo-system
spec:
  gatewayClassName: gloo-gateway
  listeners:
  - protocol: HTTP
    port: 9080
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF

kubectl rollout status -n gloo-system deployments/gloo-proxy-http
```

Get the endpoint for the gateway. Alternatively you can create a CNAME record that points to the ingress gateway and use that record for AI_LLM_DEMO_HOSTNAME:

```bash
export OPENAI_DEMO_HOSTNAME=$(kubectl get svc -n gloo-system gloo-proxy-http -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Gateway Hostname = $OPENAI_DEMO_HOSTNAME"
```

# AI Gateway Installation

Set up gateway parameters to enable AI extensions, and create AI gateway with access logging

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.gloo.solo.io/v1alpha1
kind: GatewayParameters
metadata:
  name: gloo-gateway-override
  namespace: gloo-system
spec:
  kube:
    aiExtension:
      enabled: true
    service:
      type: ClusterIP
---
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: ai-gateway
  namespace: gloo-system
  annotations:
    gateway.gloo.solo.io/gateway-parameters-name: gloo-gateway-override
spec:
  gatewayClassName: gloo-gateway
  listeners:
  - protocol: HTTP
    port: 8080
    name: http
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.solo.io/v1
kind: ListenerOption
metadata:
  name: ai-gateway-log-provider
  namespace: gloo-system
spec:
  options:
    accessLoggingService:
      accessLog:
      - fileSink:
          jsonFormat:
            httpMethod: '%REQ(:METHOD)%'
            path: '%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%'
            requestId: '%REQ(X-REQUEST-ID)%'
            responseCode: '%RESPONSE_CODE%'
            responseCodeDetails: '%RESPONSE_CODE_DETAILS%'
            systemTime: '%START_TIME%'
            targetDuration: '%RESPONSE_DURATION%'
            upstreamName: '%UPSTREAM_CLUSTER%'
            downstreamIp: '%DOWNSTREAM_LOCAL_ADDRESS%'
          path: /dev/stdout
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: ai-gateway
EOF

kubectl rollout status -n gloo-system deployments/gloo-proxy-ai-gateway
```

# AI Gateway Configuration

## OpenAI Configuration

Create a secret with the OpenAI API Key:

```bash
kubectl create secret generic openai-secret -n gloo-system --from-literal="Authorization=Bearer $OPENAI_API_KEY"
```

Create an Upstream for OpenAI with the API key:

```bash
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1
kind: Upstream
metadata:
  name: openai
  namespace: gloo-system
spec:
  ai:
    openai:
      model: "gpt-4o"
      authToken:
        secretRef:
          name: openai-secret
          namespace: gloo-system
EOF

kubectl describe upstream -n gloo-system openai
```

Create an HTTP route that maps the /openai path to the OpenAI upstream:

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: gloo-system
spec:
  parentRefs:
    - name: ai-gateway
      namespace: gloo-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /openai
    backendRefs:
    **- name: openai
      namespace: gloo-system
      group: gloo.solo.io
      kind: Upstream
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: gloo-system-routes-grant
  namespace: gloo-system
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: gloo-system
  to:
  - group: ""
    kind: Service
EOF
```

## Ollama Configuration

Deploy an Ollama instance:

```bash
kubectl apply -f- <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: ollama
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: ollama
spec:
  ports:
  - name: http
    port: 11434
    protocol: TCP
    targetPort: http
  selector:
    name: ollama
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ollama
spec:
  selector:
    matchLabels:
      name: ollama
  template:
    metadata:
      labels:
        name: ollama
    spec:
      containers:
      - env:
        - name: OLLAMA_KEEP_ALIVE
          value: "-1"
        - name: PRELOAD_MODELS
          value: qwen:0.5b,qwen:1.8b,mistral
        image: ollama/ollama:latest
        name: ollama
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - for model in $(echo $PRELOAD_MODELS | tr ',' ' '); do echo Loading $model; ollama pull "$model" ""; sleep 2; done
        ports:
        - containerPort: 11434
          name: http
          protocol: TCP
EOF

kubectl rollout status -n ollama deployments/ollama
```

Create an Upstream for Ollama

```bash
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1
kind: Upstream
metadata:
  name: ollama
  namespace: gloo-system
spec:
  ai:
    openai:
      customHost:
        host: ollama.ollama.svc.cluster.local
        port: 11434
      model: qwen:1.8b
EOF
```

## HTTP Routes

Create an HTTP route that maps /openai and /ollama paths:

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ai-gateway-route
  namespace: gloo-system
spec:
  parentRefs:
  - name: ai-gateway
    namespace: gloo-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /openai
    backendRefs:
    **- name: openai
      namespace: gloo-system
      group: gloo.solo.io
      kind: Upstream
  - matches:
    - path:
        type: PathPrefix
        value: /ollama
    backendRefs:
    - group: gloo.solo.io
      kind: Upstream
      name: ollama
      namespace: gloo-system
---
apiVersion: gateway.solo.io/v1
kind: RouteOption
metadata:
  name: ai-gateway-options
  namespace: gloo-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: ai-gateway-route
  options:
    timeout: "0"
EOF
```

# OpenAI Demo Application

Create the namespace for the OpenAI Demo application:

```bash
kubectl create namespace openai-demo
```

Create a secret for the OpenAI API key:

```bash
kubectl create secret generic openai-api-key -n openai-demo --from-literal="openai-api-key=$OPENAI_API_KEY"
```

Deploy the OpenAI Demo application:

```bash
kubectl apply -f- <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openai-demo
  namespace: openai-demo
  labels:
    app: openai-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openai-demo
  template:
    metadata:
      labels:
        app: openai-demo
    spec:
      containers:
      - name: openai-demo
        image: btjimerson/openai-demo:0.1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
          - name: OPENAI_API_KEY
            valueFrom:
              secretKeyRef:
                name: openai-api-key
                key: openai-api-key
          - name: OPENAI_BASE_URL
            value: http://gloo-proxy-ai-gateway.gloo-system.svc:8080/openai
---
apiVersion: v1
kind: Service
metadata:
  name: openai-demo
  namespace: openai-demo
  labels:
    app: openai-demo
    service: openai-demo
spec:
  ports:
  - port: 8080
    name: http
  selector:
    app: openai-demo
EOF

kubectl rollout status -n openai-demo deployments/openai-demo
```

# HTTP Route

Create an HTTP route for the demo app:

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-demo-route
  namespace: openai-demo
  labels:
    app: openai-demo
spec:
  parentRefs:
    - name: http
      namespace: gloo-system
  hostnames:
    - $OPENAI_DEMO_HOSTNAME
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /
      backendRefs:
        - name: openai-demo
          port: 8080
---
apiVersion: gateway.solo.io/v1
kind: RouteOption
metadata:
  name: openai-demo-options
  namespace: openai-demo
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: openai-demo-route
  options:
    timeout: "0"
EOF
```

Make sure you can access the pages:

```bash
open $(echo http://$OPENAI_DEMO_HOSTNAME:9080)
```

# JWT Propagation

Create 2 VirtualHostOptions (1 for the HTTP gateway and 1 for the AI gateway), that sets up the JWT provider:

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.solo.io/v1
kind: VirtualHostOption
metadata:
  name: http-jwt-provider
  namespace: gloo-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: http
  options:
    jwt:
      providers:
        selfminted:
          issuer: solo.io
          keepToken: true
          jwks:
            local:
              key: |
                -----BEGIN PUBLIC KEY-----
                MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzgEPg3jVs5HPICKB2fz2
                wUkfMIMD7GYaBhrHAQlMccneU0PWkPOctJyziMPwZTdSPIKQpZhkIa+z1FP29bbn
                hpsW0GgTLowraelvXop06IqFbHL6vHL4rewyBOV9mbQJ2NbJDYXUpk3vXgLW2mpb
                T5LAs3HzMtQmp6RMFgBjRUQmZUQI99Vx5OjnoZOEMStOzgrdhacCqvfbCrVSaYF4
                X15Hfh4A9TKQSrQhHrScWHRDYWhqVjX0dP/h7yMKrA65cjwyoPiDcP8+9PJkjU7t
                hhmly+OT46l/a/fyeqxWBe0N8SKBPyhBPbOYzDY0fsYLVl6IBGISwp50ah2ICTVS
                GQIDAQAB
                -----END PUBLIC KEY-----
---
apiVersion: gateway.solo.io/v1
kind: VirtualHostOption
metadata:
  name: ai-gateway-jwt-provider
  namespace: gloo-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: ai-gateway
  options:
    jwt:
      providers:
        selfminted:
          issuer: solo.io
          keepToken: true
          jwks:
            local:
              key: |
                -----BEGIN PUBLIC KEY-----
                MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzgEPg3jVs5HPICKB2fz2
                wUkfMIMD7GYaBhrHAQlMccneU0PWkPOctJyziMPwZTdSPIKQpZhkIa+z1FP29bbn
                hpsW0GgTLowraelvXop06IqFbHL6vHL4rewyBOV9mbQJ2NbJDYXUpk3vXgLW2mpb
                T5LAs3HzMtQmp6RMFgBjRUQmZUQI99Vx5OjnoZOEMStOzgrdhacCqvfbCrVSaYF4
                X15Hfh4A9TKQSrQhHrScWHRDYWhqVjX0dP/h7yMKrA65cjwyoPiDcP8+9PJkjU7t
                hhmly+OT46l/a/fyeqxWBe0N8SKBPyhBPbOYzDY0fsYLVl6IBGISwp50ah2ICTVS
                GQIDAQAB
                -----END PUBLIC KEY----- 
EOF
```

Test the application with no JWT token:

```bash
curl -ik http://$OPENAI_DEMO_HOSTNAME:9080/
```

Set JWT token environment variables:

```bash
export ALICE_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiAic29sby5pbyIsIm9yZyI6ICJzb2xvLmlvIiwic3ViIjogImFsaWNlIiwidGVhbSI6ICJkZXYifQ.qSCARI8Xf-7D5hWdHT71Rov-4zQGMjUz9HV04OctS6oWpTrmVBKb0JcMiiDf2rpI5NQXXk5SdLTdokC-_VXXE67EwAKMYwa4qaSFcrJIfwkOb_gSV3KqMYYYKQCCxYeHOuGaR4xdqFdMAoeGFTa7BmKWq2ZLY6c3-uWPFuW2MX1Y6SCFJXAI803FMInZcTvvjRka3WejlI-CHUw_2ZESXUf6MA0shY9aoICPjI_TrukUVoxRzu6oc0JjvcHJuqRxY-MoGberBYqWezIFlOGjWnfqvAEEp0VI-g-dMNZ7_eBFathSKD3Em7gt33T3OIDKuqkZ8i4W7WzhMIhNlSFWlA"
export BOB_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiAic29sby5pbyIsIm9yZyI6ICJzb2xvLmlvIiwic3ViIjogImJvYiIsInRlYW0iOiAib3BzIn0.M681liG4wW1DYmVwyjvAUIr4yJqZSaqODoWDSGd3egt5tuWN9ZBZLHh5odU-Y5EK8Nfq3fVzLSJtizVUWXtvMNAUUpzlfGHd99m6xdvZN9tkBWHXKTnT1vGnJ0Z9TRAlNvenSd2FZDChz7k2HW0E8IBvxMTtgPq-pMBEum2zWZIW1Bs9d8hWbEysYng7C-LdrBTj82dTps-FdPLNofigELozm8S2GQoZ5_2e42cBgngtYIcHpGJKPckPm_ZdMIujdN-5PxhLy91UX7dEI6B-O7tQyWxXV9quMEoAic67T1Np_b6ApnSXPkDspDZwUKhM6_ToiQhZqC2SwA4il9h62Q"

```

Test the application with a JWT token:

```bash
curl -ik http://$OPENAI_DEMO_HOSTNAME:9080/ -H "Authorization: Bearer $ALICE_TOKEN"
```

## Create Route Option for JWT RBAC

Create a Route Option to require a JWT token for RBAC to the AI gateway

```bash
kubectl apply -f- <<EOF
apiVersion: gateway.solo.io/v1
kind: RouteOption
metadata:
  name: openai-rbac-route-option
  namespace: gloo-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: openai
    namespace: gloo-system
  options:
    rbac:
      policies:
        viewer:
          nestedClaimDelimiter: .
          principals:
          - jwtPrincipal:
              claims:
                "org": "solo.io"
                "team": "dev"
EOF
```

Verify that Alice’s token works, but Bob’s doesn’t:

```bash
curl -ik http://$OPENAI_DEMO_HOSTNAME:9080/chat -H "Authorization: Bearer $ALICE_TOKEN" -d "prompt=How+is+your+day%3F"

curl -ik http://$OPENAI_DEMO_HOSTNAME:9080/chat -H "Authorization: Bearer $BOB_TOKEN" -d "prompt=How+is+your+day%3F"
```

# Clean up

Delete all created resources:

```bash
kubectl delete routeoption -n gloo-system openai-rbac-route-option
kubectl delete virtualhostoption -n gloo-system ai-gateway-jwt-provider
kubectl delete virtualhostoption -n gloo-system http-jwt-provider
kubectl delete httproute -n openai-demo openai-demo-route
kubectl delete service -n openai-demo openai-demo
kubectl delete deployment -n openai-demo openai-demo
kubectl delete namespace openai-demo
kubectl delete referencegrant -n gloo-system gloo-system-routes-grant
kubectl delete routeoption -n gloo-system ai-gateway-options
kubectl delete httproute -n gloo-system ai-gateway-route
kubectl delete upstream -n gloo-system ollama
kubectl delete deployment -n ollama ollama
kubectl delete service -n ollama ollama
kubectl delete namespace ollama
kubectl delete httproute -n gloo-system openai
kubectl delete upstream -n gloo-system openai
kubectl delete secret -n gloo-system openai-secret
kubectl delete listeneroption -n gloo-system ai-gateway-log-provider
kubectl delete gateway -n gloo-system ai-gateway
kubectl delete gatewayparameters -n gloo-system gloo-gateway-override
kubectl delete gateway -n gloo-system http

helm uninstall -n gloo-system gloo
kubectl get crd -A | grep 'solo' | xargs kubectl delete crd
kubectl delete namespace gloo-system

unset AI_LLM_DEMO_HOSTNAME
unset ALICE_TOKEN
unset BOB_TOKEN
```