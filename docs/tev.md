# Deploy Tev1-4B

[Overview](index.md) · [API examples](decision-api.md) · [Operations](operations.md)

Use Tev when bilingual relevance and yes/no quality matter most. The checkpoint is experimental. Its underlying multimodal model does not imply image support through this version of the decision endpoint: all eight image requests were rejected in our run. Use d1 for images on this API.

## Tested deployment

| Item | Value |
|---|---|
| Model | `togethercomputer/Tev1-4B-experimental` |
| Revision | `0b7becf017daa0e5eb222f8ce7483c8c8259c52f` |
| Image | `vllm/vllm-openai:v0.32.0` |
| Precision | BF16 |
| GPU tested | One NVIDIA L40S |
| Native decision endpoint | `POST /v1/systemone` |
| Server context limit | 12,288 tokens |
| Max scheduled sequences | 16 |

## Prerequisites

Use a Linux host with Docker and NVIDIA Container Toolkit configured so containers can see the GPU. Check `nvidia-smi` on the host. Provide network access to download the image and model, and writable disk for the cache. For an offline host, stage the exact image and pinned model snapshot beforehand; these commands assume online downloads on first use.

The L40S configuration is measured. A 24 GB L4 is a smaller-GPU candidate to validate, not a certified replacement for the 12k-context, 16-sequence configuration. Parameter-count weight estimates exclude KV cache, vision components, activations and runtime allocations. See [memory tuning](operations.md#memory-and-throughput).

## Start the server

Run one guide at a time: both examples use host port 8001 and reserve 87% of GPU memory. The named volume survives container removal.

```bash
docker volume create decision-model-cache

docker run --detach \
  --name tev-decisions \
  --gpus all \
  --ipc=host \
  --publish 127.0.0.1:8001:8001 \
  --mount source=decision-model-cache,target=/model-cache \
  --entrypoint python3 \
  vllm/vllm-openai:v0.32.0 \
  -m vllm.entrypoints.openai.api_server \
  --model togethercomputer/Tev1-4B-experimental \
  --revision 0b7becf017daa0e5eb222f8ce7483c8c8259c52f \
  --dtype bfloat16 \
  --max-model-len 12288 \
  --max-num-seqs 16 \
  --gpu-memory-utilization 0.87 \
  --host 0.0.0.0 \
  --port 8001 \
  --download-dir /model-cache \
  --logprobs-mode raw_logprobs \
  --no-enable-prefix-caching \
  --enforce-eager
```

The process listens on all container interfaces so Docker port forwarding works; Docker publishes it only on the host's loopback interface. No API key is configured in this local example. For network access, use the [protected access pattern](operations.md#access-and-secrets).

`--logprobs-mode raw_logprobs` preserves the probability diagnostics used by this API. Eager mode and disabled prefix caching reproduce the evaluation settings; they are a baseline, not a universal optimum.

## Check readiness

```bash
docker logs --tail 100 tev-decisions
curl --fail-with-body http://127.0.0.1:8001/health
curl --fail-with-body http://127.0.0.1:8001/v1/models
```

Wait for startup to finish before making decisions. Cold starts in our run were approximately 132 seconds, including this host's download/load path; startup time varies with cache and host.

## Smoke test

```bash
curl --fail-with-body --silent --show-error \
  http://127.0.0.1:8001/v1/systemone \
  -H 'Content-Type: application/json' \
  --data-binary @- <<'JSON'
{
  "model": "togethercomputer/Tev1-4B-experimental",
  "state": "The quarterly report was released on 15 October.",
  "questions": {
    "released": {
      "type": "noul",
      "instructions": "Was the quarterly report released?"
    }
  },
  "chat_template_kwargs": {"enable_thinking": false}
}
JSON
```

Expect HTTP 200, `object: "structured_decision"`, and `answers.released.noul` between 0 and 1. A value at least 0.5 means a yes decision under a simple threshold. Do not require an exact numeric probability for this example.

## Next steps

Use [score and house-category examples](decision-api.md) or the [email-search pipeline](decision-api.md#email-search-pipeline). Keep the exact model ID in each request; do not rename the deployment when reproducing the baseline.

## Stop and remove

```bash
docker stop tev-decisions
docker rm tev-decisions
```

This removes the process and container while retaining the cache volume. On a rented GPU host, also destroy the cloud instance through its provider and verify absence. Removing a Docker container does not end the rental.

## Sources

- [Model card](https://huggingface.co/togethercomputer/Tev1-4B-experimental)
- [vLLM 0.32.0 decision API](https://docs.vllm.ai/en/v0.32.0/serving/online_serving/structured_decisions/)
- Exact revision and CLI flags: saved evaluation `results/manifest.json`, linked from the [overview](index.md).
