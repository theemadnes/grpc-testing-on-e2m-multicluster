# grpc-testing-on-e2m-multicluster

I have a multi-region setup using 2 x GKE Autopilot clusters with ASM deployed using E2E encryption (i.e. TLS from GFE to ingress gateway) and want to verify that gRPC services work. The clusters were configured using [this](https://github.com/theemadnes/multi-cluster-gateway-asm-01) repo.

We'll use [whereami](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/main/whereami) in gRPC mode as the sample application, with two instances (`frontend` and `backend`) deployed to both clusters.

### setup

```
export CLUSTER_1_CTX=autopilot-cluster-us-central1 # replace with your kube contexts
export CLUSTER_2_CTX=autopilot-cluster-us-east4

# create namespaces and label them for sidecar injection
kubectl --context=$CLUSTER_1_CTX create namespace whereami-grpc-frontend
kubectl --context=$CLUSTER_2_CTX create namespace whereami-grpc-frontend
kubectl --context=$CLUSTER_1_CTX create namespace whereami-grpc-backend
kubectl --context=$CLUSTER_2_CTX create namespace whereami-grpc-backend

kubectl --context=$CLUSTER_1_CTX label namespace whereami-grpc-frontend istio-injection=enabled
kubectl --context=$CLUSTER_2_CTX label namespace whereami-grpc-frontend istio-injection=enabled
kubectl --context=$CLUSTER_1_CTX label namespace whereami-grpc-backend istio-injection=enabled
kubectl --context=$CLUSTER_2_CTX label namespace whereami-grpc-backend istio-injection=enabled

# deploy whereami-grpc backend
mkdir -p whereami-grpc-backend/base

cat <<EOF > whereami-grpc-backend/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s-grpc
EOF

mkdir whereami-grpc-backend/variant

cat <<EOF > whereami-grpc-backend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami-grpc
data:
  BACKEND_ENABLED: "False" # assuming you don't want a chain of backend calls
  METADATA:        "backend"
EOF

cat <<EOF > whereami-grpc-backend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami-grpc"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami-grpc-backend/variant/kustomization.yaml 
nameSuffix: "-backend"
namespace: whereami-grpc-backend
commonLabels:
  app: whereami-grpc-backend
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=${CLUSTER_1_CTX} apply -k whereami-grpc-backend/variant
kubectl --context=${CLUSTER_2_CTX} apply -k whereami-grpc-backend/variant

# deploy whereami-grpc frontend
mkdir -p whereami-grpc-frontend/base

cat <<EOF > whereami-grpc-frontend/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s-grpc
EOF

mkdir whereami-grpc-frontend/variant

cat <<EOF > whereami-grpc-frontend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami-grpc
data:
  BACKEND_ENABLED: "True"
  BACKEND_SERVICE:        "http://whereami-grpc-backend.whereami-grpc-backend.svc.cluster.local"
EOF

cat <<EOF > whereami-grpc-frontend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami-grpc"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami-grpc-frontend/variant/kustomization.yaml 
nameSuffix: "-frontend"
namespace: whereami-grpc-frontend
commonLabels:
  app: whereami-grpc-frontend
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=${CLUSTER_1_CTX} apply -k whereami-grpc-frontend/variant
kubectl --context=${CLUSTER_2_CTX} apply -k whereami-grpc-frontend/variant
```