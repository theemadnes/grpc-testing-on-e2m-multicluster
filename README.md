# grpc-testing-on-e2m-multicluster

I have a multi-region setup using 2 x GKE Autopilot clusters with ASM deployed using E2E encryption (i.e. TLS from GFE to ingress gateway) and want to verify that gRPC services work. The clusters were configured using [this](https://github.com/theemadnes/multi-cluster-gateway-asm-01) repo.

We'll use [whereami](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/main/whereami) in gRPC mode as the sample application, with two instances (`frontend` and `backend`) deployed to both clusters.

### setup

