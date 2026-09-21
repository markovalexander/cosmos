<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: OpenMDW-1.1 -->

# Deploy Cosmos3 Reasoner with Dynamo

Run a Dynamo frontend with Cosmos3 Reasoner workers on Kubernetes. This
example uses the NGC frontend image and `python3 -m dynamo.frontend` for
Chat Completions, with round-robin routing and session affinity. It starts
one Nano BF16 worker on one GPU, with persistent video sessions disabled.
For those sessions, use [Dynamo with streaming](dynamo_with_streaming.md).

The plain YAML follows `cosmos3/dynamo/helm/cosmos3-dynamo` in
`cosmos-genai-nim`. Helm is not needed; the Dynamo operator is still required.
Replace values in `<angle-brackets>` with your own values before applying.

Set `<nim-image-build>` in [`reasoner.yaml`](reasoner.yaml) to the available
Cosmos3 NIM image address, including its tag or digest. The manifest retains
the source chart's pinned NGC frontend and `1.5.0` operator runtime version.

## Prerequisites

- A Kubernetes cluster with NVIDIA GPU support and a compatible Dynamo
  operator/platform serving `nvidia.com/v1beta1` `DynamoGraphDeployment` resources.
- Capacity for each worker: one GPU compatible with the image's Nano BF16
  profile, 16 CPUs, and 96 GiB of pod memory. The frontend requests another
  4 CPUs and 8 GiB. These are chart resource settings, not GPU memory minima;
  verify the selected image's profile against your hardware.
- A ReadWriteMany (RWX) storage class and a 200 GiB model cache writable by
  UID/GID 1000. Workers must be able to download model artifacts.
- `kubectl`, cURL, Python 3, and an exported `NGC_API_KEY` with image and model
  access; see [Network and NGC access](../prerequisites.md#network-and-ngc-access).

## Prepare the namespace and cache

Run commands from `cookbooks/cosmos3/nim/dynamo`, with the intended Kubernetes
context selected. Create the namespace and the two secrets referenced by the
manifest:

```bash
kubectl create namespace cosmos3
kubectl -n cosmos3 create secret docker-registry ngc-registry-secret \
  --docker-server=nvcr.io --docker-username='$oauthtoken' \
  --docker-password="$NGC_API_KEY"
kubectl -n cosmos3 create secret generic ngc-api-secret \
  --from-literal=NGC_API_KEY="$NGC_API_KEY"
```

Reuse these resources if they already exist. In
[`model-cache.yaml`](model-cache.yaml), replace
`<your-rwx-storage-class>` with your cluster's RWX storage class, then apply it:

```bash
kubectl -n cosmos3 apply -f model-cache.yaml
```

If you already have a suitable cache PVC, skip its creation and set
`claimName` in [`reasoner.yaml`](reasoner.yaml) to
`<existing-cache-pvc>` (your claim's name).
The cache mounts at `/opt/nim/.cache` without a subdirectory. A storage class
using `WaitForFirstConsumer` can leave the PVC Pending until workers start.

## Deploy

```bash
kubectl -n cosmos3 apply -f reasoner.yaml
kubectl -n cosmos3 get pods \
  -l nvidia.com/dynamo-graph-deployment-name=cosmos3-reasoner -w
```

## Check the deployment

Wait for the frontend and worker pods to become Ready, then stop the watch
with Ctrl+C. Cold startup includes model download and warmup. The operator
creates the frontend Service; forward it in a separate terminal:

```bash
kubectl -n cosmos3 port-forward service/cosmos3-reasoner-frontend 8000:8000
```

Check registration and discover the served model:

```bash
export NIM_URL=http://localhost:8000
curl -fsS "$NIM_URL/health" | python3 -m json.tool
curl -fsS "$NIM_URL/v1/models" | python3 -m json.tool
```

Before sending requests, confirm `/health` lists a distinct `generate`
instance for each worker and `/v1/models` is nonempty. Frontend health alone
does not establish worker readiness. Workers expose NIM's
`/v1/health/ready`, `/v1/metadata`, and `/v1/metrics` on their own port 8000;
port-forward a worker pod to a different local port to inspect those endpoints.
Check each worker's readiness and selected profile there.

## Send a request

The frontend serves `/v1/chat/completions`. Discover the model ID rather than
assuming it matches the Kubernetes resource name:

```bash
MODEL=$(curl -fsS "$NIM_URL/v1/models" \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["data"][0]["id"])')
curl -fsS "$NIM_URL/v1/chat/completions" \
  -H 'Content-Type: application/json' \
  -d "{\"model\":\"$MODEL\",\"messages\":[{\"role\":\"user\",\"content\":\"Describe how a robot can avoid obstacles.\"}],\"max_tokens\":128}"
```

See [Reasoning](../reasoning.md) for image/video request formats. The frontend
uses Dynamo's API, including OpenAI `response_format`; NIM-specific aliases
are handled by the worker's NIM proxy. Cookbook clients that require
`/v1/metadata` must target a worker's NIM HTTP endpoint, not the frontend.

## Scale or stop

To add workers, set `ReasonerWorker.replicas` to `<worker-count>` in
[`reasoner.yaml`](reasoner.yaml) and apply it again. Each
replica needs the listed resources and access to the shared cache. Changing
the model or GPU topology also requires matching `NIM_MODEL_VARIANT`,
`NIM_PRECISION`, `NIM_TAGS_SELECTOR`, and GPU requests/limits to the selected
image's profile. This example uses fixed replicas; autoscaling is not enabled.

To stop serving, delete only the graph deployment:

```bash
kubectl -n cosmos3 delete -f reasoner.yaml
```

This removes the frontend and workers while retaining the cache PVC and
secrets for reuse. Stop the port-forward with Ctrl+C.
