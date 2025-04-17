# kgateway and Argo Rollouts

Use Argo Rollouts and kgateway for canary deployments.

## Prerequisites

To get started make sure you have the following:

- A Kubernetes cluster with cluster admin access. You can also create a local cluster with [k3s](https://k3s.io/) or [kind](https://kind.sigs.k8s.io/).
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) CLI with the [argo plugin](https://argo-rollouts.readthedocs.io/en/stable/features/kubectl-plugin/).
- [Helm](https://helm.sh/docs/intro/install/) CLI.
- The following, which typically exist on most Linux and MacOS systems already:
    - curl
    - watch

## kgateway installation

Install the Kubernetes Gateway CRDs and kgateway:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

helm upgrade -i --create-namespace --namespace kgateway-system --version v2.0.0 kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds
helm upgrade -i --namespace kgateway-system --version v2.0.0 kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway
```

## Argo Rollouts installation

Install Argo Rollouts:

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

kubectl rollout status -n argo-rollouts deploy argo-rollouts
```

Install the Gateway API plugin:

```bash
kubectl replace -f- <<EOF
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argo-rollouts-config
  namespace: argo-rollouts
data:
  trafficRouterPlugins: |-
    - name: "argoproj-labs/gatewayAPI"
      location: "https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/releases/download/v0.6.0/gatewayapi-plugin-linux-amd64"
EOF
```

<aside>
💡

You can view the full list of releases and architectures for the Gateway API plugin here:

[https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/releases/](https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/releases/)

</aside>

Create a role to allow Argo to modify HTTP Routes:

```bash
kubectl apply -f- <<EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gateway-controller-role
  namespace: argo-rollouts
rules:
  - apiGroups:
      - gateway.networking.k8s.io
    resources:
      - httproutes
    verbs:
      - get
      - patch
      - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gateway-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: gateway-controller-role
subjects:
  - namespace: argo-rollouts
    kind: ServiceAccount
    name: argo-rollouts
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: argo-rollouts-configmap-access
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: default
  name: argo-rollouts-configmap-binding
subjects:
- kind: ServiceAccount
  name: argo-rollouts
  namespace: argo-rollouts
roleRef:
  kind: Role
  name: argo-rollouts-configmap-access
  apiGroup: rbac.authorization.k8s.io
EOF
```

Restart Argo Rollouts:

```bash
kubectl rollout restart deployment -n argo-rollouts argo-rollouts

kubectl rollout status -n argo-rollouts deploy argo-rollouts
```

## Application Setup

Create a gateway HTTP listener:

```bash
kubectl apply -f- <<EOF
---
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: http-gateway
spec:
  gatewayClassName: kgateway
  listeners:
  - name: http
    protocol: HTTP
    port: 80
EOF
```

Set the address for the gateway as an environment variable:

```bash
export GATEWAY_ADDRESS=$(kubectl get svc http-gateway -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo "Gateway address: $GATEWAY_ADDRESS"
```

Install the ratings service that the reviews service calls:

```bash
kubectl apply -f- <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: ratings
  labels:
    app: ratings
    service: ratings
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: ratings
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-ratings
  labels:
    account: ratings
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ratings-v1
  labels:
    app: ratings
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ratings
      version: v1
  template:
    metadata:
      labels:
        app: ratings
        version: v1
    spec:
      serviceAccountName: bookinfo-ratings
      containers:
      - name: ratings
        image: docker.io/istio/examples-bookinfo-ratings-v1:1.20.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
EOF
```

Create canary and stable reviews services for Argo to use:

```bash
kubectl apply -f- <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: reviews-stable
  labels:
    app: reviews
    service: reviews
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: reviews
---
apiVersion: v1
kind: Service
metadata:
  name: reviews-canary
  labels:
    app: reviews
    service: reviews
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: reviews
EOF
```

Create an HTTPRoute that references the canary and stable services:

```bash
kubectl apply -f- <<EOF
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo-reviews
spec:
  parentRefs:
  - name: http-gateway
  hostnames:
  - $GATEWAY_ADDRESS
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /reviews
    backendRefs:
    - name: reviews-canary
      port: 9080
    - name: reviews-stable
      port: 9080
EOF
```

## Rollouts

Create a Rollout for the reviews service, instead of a traditional deployment:

```bash
kubectl apply -f- <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-reviews
  labels:
    account: reviews
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: reviews-rollout
  labels:
    app: reviews
spec:
  replicas: 1
  strategy:
    canary:
      stableService: reviews-stable
      canaryService: reviews-canary
      trafficRouting:
        managedRoutes:
        - name: qa-header
        plugins:
          argoproj-labs/gatewayAPI:
            # Use the bookinfo-reviews HTTPRoute for header
            # information
            httpRoutes:
            - name: bookinfo-reviews
              useHeaderRoutes: true
            namespace: default
      steps:
      # Set the canary to 1 replica
      - setCanaryScale:
          replicas: 1
      # Allow any request with the role=qa header
      # to test the canary
      - setHeaderRoute:
          name: qa-header
          match:
          - headerName: role
            headerValue:
              exact: qa
      # Pause indefinitely. This requires a human to say
      # OK before proceeding
      - pause: {}
      # Gradually shift traffic to the canary
      - setWeight: 10
      - pause: {duration: 10s}
      - setWeight: 20
      - pause: {duration: 10s}
      - setWeight: 40
      - pause: {duration: 10s}
      - setWeight: 60
      - pause: {duration: 10s}
      - setWeight: 100
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: reviews
  template:
    metadata:
      labels:
        app: reviews
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
        imagePullPolicy: IfNotPresent
        env:
        - name: LOG_DIR
          value: "/tmp/logs"
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: wlp-output
          mountPath: /opt/ibm/wlp/output
      volumes:
      - name: wlp-output
        emptyDir: {}
      - name: tmp
        emptyDir: {}
EOF
```

Watch the rollout’s status in a separate window:

```bash
watch --color "kubectl argo rollouts get rollout reviews-rollout"
```

Make sure the reviews service is working:

```bash
curl $(echo "http://$GATEWAY_ADDRESS/reviews/123") | jq
```

Create a rollout for version 2 of the reviews service:

```bash
kubectl argo rollouts set image reviews-rollout reviews=docker.io/istio/examples-bookinfo-reviews-v2:1.20.2
```

Check the stable and canary release services select different pods:

```bash
kubectl get service -o wide -l app=reviews
```

Send a request to the canary service by setting the 'role=qa' header; the pod in the response should be the canary pod:

```bash
curl -H "role: qa" $(echo "http://$GATEWAY_ADDRESS/reviews/123") | jq
```

Send a request to the canary service without the header value; the pod in the response should be the stable pod:

```bash
curl $(echo "http://$GATEWAY_ADDRESS/reviews/123") | jq
```

Promote the canary and view the rollout status window for progression:

```bash
kubectl argo rollouts promote reviews-rollout
```