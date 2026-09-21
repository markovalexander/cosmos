<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: OpenMDW-1.1 -->

# Cosmos3 Reasoner with Dynamo

Choose a deployment:

| Guide | Manifest | Frontend and behavior |
| --- | --- | --- |
| [Regular Dynamo](dynamo_deployment.md) | [reasoner.yaml](reasoner.yaml) | Pinned `nvcr.io` frontend running `dynamo.frontend`; Chat Completions, no persistent video sessions. |
| [Dynamo with streaming](dynamo_with_streaming.md) | [reasoner_with_streaming.yaml](reasoner_with_streaming.yaml) | Same NIM image for frontend and workers; frontend runs `streaming.dynamo.frontend` for Chat Completions plus REST video sessions. |

Both use [model-cache.yaml](model-cache.yaml) and require the Dynamo operator.
Use the available Cosmos3 NIM image address for `<nim-image-build>` in both
variants; streaming uses that same image for the frontend and workers.
Replace `<angle-bracket>` placeholders before applying. These are alternative
manifests for the same `cosmos3-reasoner` deployment; apply only one. Switching
variants replaces pods and loses active streaming sessions.

Here, **streaming** means persistent video sessions, not token streaming in a
Chat Completions response. The Dynamo video-session path uses REST only.
