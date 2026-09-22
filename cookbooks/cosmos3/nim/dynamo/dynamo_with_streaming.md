<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: OpenMDW-1.1 -->

# Deploy Cosmos3 Reasoner with Dynamo streaming

Use [`reasoner_with_streaming.yaml`](reasoner_with_streaming.yaml) for Chat
Completions plus persistent video sessions over REST. The frontend and workers
use the same Cosmos3 NIM image, with different launch commands. The frontend
runs the bundled streaming adapter; workers run the NIM server with streaming
enabled.

> **WebSocket is not supported in Dynamo mode.** The
> `WS /v1/streaming/ws` endpoint is unavailable on both the shared frontend
> and worker NIM endpoints. Use the REST session API below.

First complete the [prerequisites](dynamo_deployment.md#prerequisites) and
[namespace and cache setup](dynamo_deployment.md#prepare-the-namespace-and-cache).
If reusing a cache, set `claimName` in the streaming YAML to
`<existing-cache-pvc>`. Replace all `<angle-bracket>` values before applying.

## Set the NIM image

In [`reasoner_with_streaming.yaml`](reasoner_with_streaming.yaml), replace
both occurrences of `<nim-image-build>` with the same available Cosmos3 NIM
image address, including its tag or digest. The image must include
`/opt/nim/streaming/dynamo/`.

The frontend runs `/opt/nim/.venv/bin/python -m streaming.dynamo.frontend`
from `/opt/nim`. It serves REST sessions and forwards ordinary Chat Completions
to Dynamo without loading a model or requesting a GPU. The stock
`python3 -m dynamo.frontend` command does not expose the session routes.

Workers start `/opt/nim/start_server.sh`: `NIM_DYNAMO_WORKER=true` and
`NIM_ENABLE_STREAMING=1` select the streaming worker and its private frontend.
See [Dynamo worker configuration](../configuration.md#run-nim-as-a-dynamo-worker)
and [Dynamo streaming configuration](../configuration.md#dynamo-streaming)
for environment variables, defaults, precedence, and timeouts.
The source pins Dynamo `1.5.0.dev20260902`; the graph's operator runtime
override remains `1.5.0`.

The YAML sets synchronous scheduling explicitly and uses the same
`NIM_DYNAMO_STREAMING_ENDPOINT` on frontend and workers. Keep that value
identical; use a distinct endpoint for each separate model pool.

## Deploy and check

Run from `cookbooks/cosmos3/nim/dynamo`. If your image uses another registry,
configure its credentials and replace both `imagePullSecrets` entries with
`<your-image-pull-secret>`.

```bash
kubectl -n cosmos3 apply -f reasoner_with_streaming.yaml
kubectl -n cosmos3 get pods \
  -l nvidia.com/dynamo-graph-deployment-name=cosmos3-reasoner -w
```

Both YAML files define `cosmos3-reasoner`: apply only the chosen variant.
Applying the other variant updates that deployment and replaces its pods;
existing streaming sessions are lost.

Follow the shared [readiness and port-forward checks](dynamo_deployment.md#check-the-deployment),
including setting `NIM_URL`. Check frontend and worker logs for
`[Dynamo streaming]` messages confirming that the HTTP adapter and worker REST
handlers attached. Chat Completions use the same
[request example](dynamo_deployment.md#send-a-request).

## Stream video frames over REST

Use the forwarded frontend at `NIM_URL`. Create a session, send one JPEG or
PNG frame at a time, then delete the session. This Dynamo path supports raw
image bodies and JSON `image_b64`. The `/v1/streaming/config` route is also
unavailable, so the cookbook's `examples/streaming.py` client cannot be used
here, even with `--transport rest`: its preflight requires that config route.

After the readiness checks above, try one local JPEG frame:

```bash
(
  set -euo pipefail
  created=$(curl -fsS "$NIM_URL/v1/streaming/sessions" \
    -H 'Content-Type: application/json' \
    -d '{"system_prompt":"Describe each frame in one short sentence.","question":"What is happening?","fps":1,"sampling":{"max_tokens":48}}')
  session_id=$(printf '%s' "$created" | python3 -c \
    'import json,sys; print(json.load(sys.stdin)["session_id"])')
  trap 'curl -fsS -X DELETE "$NIM_URL/v1/streaming/sessions/$session_id"' EXIT

  curl -fsS "$NIM_URL/v1/streaming/sessions/$session_id/frame" \
    -H 'Content-Type: image/jpeg' --data-binary '@<path-to-frame.jpg>'
)
```

For video, decode frames on the client and wait for each reply before sending
the next. Keep the returned `session_id` unchanged: frame uploads and deletion
route to its owner even through a different frontend replica. This ownership
does not depend on the Chat Completions affinity header or its 300-second TTL.
See [Streaming](../streaming.md#session-configuration) for session fields and
[streaming settings](../configuration.md#streaming-video-sessions) for limits;
the REST idle timeout defaults to 120 seconds.

A session-route 404 can indicate an old frontend/worker image or streaming
disabled on the worker; an existing session can also expire. A 503 can mean
workers are loading, capacity is full, or the owning worker is unavailable.
Sessions do not migrate after worker loss; create a new one. Do not
automatically retry a frame after a timeout.

## Scale or stop

Set `ReasonerWorker.replicas` to `<worker-count>` in the streaming YAML and
apply that file again. Each replica needs the shared
[resources and cache](dynamo_deployment.md#prerequisites). Scaling down or
replacing workers loses their active sessions; session state does not migrate.
Set the count to `0` to release the GPUs while retaining the frontend and cache;
restore a positive count and wait for readiness before sending more requests.

To stop serving while keeping the cache PVC and secrets:

```bash
kubectl -n cosmos3 delete -f reasoner_with_streaming.yaml
```

Stop the port-forward with Ctrl+C.
