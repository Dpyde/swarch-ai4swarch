# System under study: Online Boutique (trimmed snapshot)

`online-boutique/` is a snapshot of [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) at commit `38e7348eb289eb5b87c0c6e8cb19ced0449dc389`, licensed under the Apache License 2.0 (see [`online-boutique/LICENSE`](online-boutique/LICENSE)).

It is an e-commerce demo: users browse products, add them to a cart, and check out.

## Changes made for this exercise

To keep the exercise focused, and Copilot's context small, the following were **removed**:

- the upstream `README.md` and `docs/`, including the official architecture diagram (recovering it is your job in Task 1)
- deployment variants: `helm-chart/`, `kustomize/`, `istio-manifests/`, `terraform/`, `release/`, build configs
- generated protobuf code (`genproto/`, `*_pb2*.py`) and lock files
- the frontend's static assets (images, CSS)

Kept: all service source code, `protos/`, and the per-service `kubernetes-manifests/`. No source file was modified.

The system is **not meant to be run** in this exercise. You analyse it, you don't deploy it.
