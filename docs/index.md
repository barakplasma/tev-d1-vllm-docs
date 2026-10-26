# Tev and d1 deployment with vLLM

Deploy the exact model revisions that completed our 26 October 2026 evaluation using the stock `vllm/vllm-openai:v0.32.0` image. These guides require no server patches or package changes.

- [Deploy Tev1-4B](tev.md)
- [Deploy LiquidAI d1-3B](d1.md)
- [Decision API examples](decision-api.md)
- [Operations and troubleshooting](operations.md)
- [Build and host with Chainguard Hugo](hugo.md)

## Choose a model

| Requirement | Starting choice | Evidence from this evaluation |
|---|---|---|
| Bilingual English/Hebrew relevance and yes/no | Tev1-4B | Higher relevance and yes/no accuracy on the frozen fixtures |
| Lower latency and cost per decision | d1-3B | About twice the short-task throughput on the same L40S |
| Images through `/v1/systemone` | d1-3B | Accepted 8/8 image requests; Tev rejected 8/8 |
| Reliable latest-client-email selection | Either, with the pipeline below | Both 4/4 with metadata filtering and semantic classification |

Tev is a Qwen-based decision fine-tune; d1 uses an LFM vision-language backbone. These are two specific deployments, not interchangeable guarantees for every Jev-inspired checkpoint.

## Measured baseline

One NVIDIA L40S with 46,068 MiB visible VRAM; BF16; eager execution; prefix caching disabled; client on the GPU host. Each model ran separately. The observed GPU-plus-disk rental rate was $0.47/hour; this is historical, not a current offer.

| Metric | Tev1-4B | d1-3B | Direction |
|---|---:|---:|---|
| Median short-task E2E, concurrency 1 | 50.1 ms | 19.5 ms | Lower is better |
| Short decisions/s, concurrency 8 | 65.1 | 133.2 | Higher is better |
| Input tokens/s, same load | 13,086 | 25,037 | Higher is better; tokenizers differ |
| Estimated USD / million decisions | 2.01 | 0.98 | Lower is better |
| Relevance nDCG@5, English / Hebrew | 0.960 / 0.939 | 0.940 / 0.897 | Higher is better |
| Yes/no accuracy, English / Hebrew | 100% / 100% | 87.5% / 75% | Higher is better |

The throughput windows were short screening runs. Cost assumes continuous measured throughput and excludes downloads, startup and idle time. The 12,288-token setting is the tested server limit, not a claim about the model's maximum context. These figures do not predict L4, A100 or consumer-GPU performance.

Both models scored 20/20 in each language on explicit house-trait descriptions with the improved rubric. Named-character house recall was less reliable. The image sample establishes request admission, not comprehensive visual accuracy.

## Documentation format

Copy this `docs/` directory into a Markdown documentation generator. Pages use ordinary Markdown, fenced code blocks and relative `.md` links. They require no MDX, proprietary components, plugins or front matter. Configure navigation in your generator if needed. Use [the included Hugo setup](hugo.md) for a distroless build and runtime. The root `mkdocs.yml` remains an optional alternative for MkDocs.

## Provenance

- [Saved evaluation report](https://chatgpt.com/library?file_id=libfile_41c223cead448191b34041a5cdb03ea6&account_id=personal)
- [Raw results and reproduction bundle](https://chatgpt.com/library?file_id=libfile_4449787fce7081919e38587d5a80c480&account_id=personal)
- [Versioned vLLM decision API documentation](https://docs.vllm.ai/en/v0.32.0/serving/online_serving/structured_decisions/)

Commands below reproduce the tested serving flags, with Docker host-loopback publishing and a persistent cache added for deployment. This documentation was checked against the saved commands and official pages on 26 October 2026; no new GPU was rented to write it.
