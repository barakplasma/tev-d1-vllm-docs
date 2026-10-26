# Operations and troubleshooting

[Overview](index.md) · [Tev](tev.md) · [d1](d1.md) · [API examples](decision-api.md)

## Access and secrets

The deployment commands publish only to host loopback. For a remote rented machine, an SSH tunnel is a simple access method:

```bash
ssh -N -L 8001:127.0.0.1:8001 user@GPU_HOST
```

Then use `http://127.0.0.1:8001` on your workstation. Replace `user@GPU_HOST` with your provider's actual SSH connection details and port settings. Put an authenticated TLS reverse proxy in front of the service if a shared network endpoint is required. Keep model/API credentials out of Markdown files and source control.

For private or gated model access, supply an authorized Hugging Face token via your deployment's secret mechanism. The tested commands needed no `--trust-remote-code` flag and did not install model-card snippets into the image.

## Memory and throughput

Start with the exact L40S settings. On a smaller GPU, first reduce `--max-model-len` and `--max-num-seqs`, then rerun your longest text and image cases. Examples such as context 4096 and max sequences 4 are tuning candidates, not configurations measured in this evaluation. Reducing GPU-memory utilization leaves more headroom for other workloads but reduces vLLM's usable budget; it is not a universal OOM fix.

Run only one example container on a GPU at a time. To change flags, stop and remove the container, then recreate it with the edited command. Do not launch a second 87%-memory server alongside the first.

After the baseline works, test disabling eager mode or enabling prefix caching separately. Record each change and rerun latency, quality and memory measurements. Repeated shared states can behave differently from unique single-question requests. No performance claim in these guides includes those tuning changes or quantization.

## Common failures

| Symptom | First check |
|---|---|
| Container cannot access a GPU | Host `nvidia-smi`, NVIDIA Container Toolkit, `--gpus all` |
| Connection refused | Startup logs, host port mapping, process readiness, SSH tunnel |
| HTTP 404 on `/v1/systemone` | The running image/version and exact URL; chat availability alone is insufficient |
| HTTP 501 | Actual loaded architecture and supported decision strategy; keep the model ID and revision pinned |
| HTTP 400 | Read the body: question fields/types, labels, context length, or unsupported media |
| Image request rejected on Tev | Use d1 for images on this tested API/version |
| Out-of-memory | Other GPU processes; context, concurrency and image size; retest a smaller configuration |
| Downloads fail | Egress, model authorization, cache permissions and available disk |
| Similar input gives different probabilities | Batch/cache/numerical effects; measure stability on your deployment |
| High relative answer probability but poor decisions | Inspect rubric, label mass, missing-evidence handling and validation coverage |

Keep the full HTTP error body. Do not treat failures as false/no or as zero model accuracy.

## Measure deployment performance

Warm up after startup and report cold-start time separately. Use a representative mix of short and long inputs; include images only for d1. Measure concurrency 1 and your expected production concurrency. Log at least:

- Model ID/revision, image tag or digest, flags, GPU and driver.
- Request ID, question count, input/output usage and success/error status.
- Wall-clock E2E per request, p50/p95, successful requests/s and successful decisions/s.
- Accuracy/ranking quality on held-out labeled tasks, language and input-length groups.
- Whole-workflow elapsed time for concurrent email search, without summing overlapping candidate latencies.

Use `time.perf_counter()` around the request. A client on another continent adds network time; our latency baseline used a co-located client. For these first-token decisions, E2E is the meaningful latency metric. Separately stream `/v1/chat/completions` if explanation-generation TTFT or decoder speed matters.

At hourly rental rate `R` and sustained throughput `D` decisions/second:

```text
steady USD per million decisions = R * 1,000,000 / (3600 * D)
exclusive GPU cost of a workflow = R * workflow_wall_seconds / 3600
```

These estimates exclude startup, idle time and transfers and do not allocate a shared service's costs between tenants.

## Reproducibility and updates

Pin image version and model revision together. The supplied commands use the version tag that was tested; for stricter image reproducibility, record the pulled image digest and deploy by that digest. No verified digest was retained in the benchmark, so none is invented here.

```bash
docker image inspect vllm/vllm-openai:v0.32.0 --format '{{json .RepoDigests}}'
```

Do not silently substitute a newer `stable` image. Validate the pinned release first, then smoke-test and benchmark any upgrade as a separate deployment.

## Preserve results and end rentals

Before shutdown, export raw data and verify a downloaded copy with a checksum. Use checkpoint filenames that include the model and revision. Do not depend on the terminal/controller connection remaining alive: in the original four-model campaign the export controller hit a broken pipe after preserving Tev and d1, leaving no recovered later-model results.

Use a provider-side budget deadline in addition to a local process. A five-minute follow-up window is idle rental time and belongs in the budget. Delete the exact intended cloud instance and verify its absence; Docker stop or a cloud stop action may leave billable storage or a retained rental. Never reuse instance IDs from the historical benchmark.

## Sources

- [vLLM 0.32.0 decision API](https://docs.vllm.ai/en/v0.32.0/serving/online_serving/structured_decisions/)
- [Evaluation and limitations](index.md#provenance)
