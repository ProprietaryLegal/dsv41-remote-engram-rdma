# DeepSeek-V4.1-Flash on DGX Spark / GB10: remote Engram tables in x86 RAM over RDMA

Serve the model's Engram lookup rows from an x86 host instead of each Spark's NVMe. The engine keeps its local reader as an exact fallback. This source package is part of a system with two DGX Spark (GB10) nodes and an x86 offload host with RTX 3090s; Engram itself uses host RAM and no x86 GPU.

**Hugging Face:** [Card and files](https://huggingface.co/ProprietaryLegal/dsv41-remote-engram-rdma) · **Stack overview:** [GitHub](https://github.com/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) · **Collection:** [All entries](https://huggingface.co/collections/ProprietaryLegal/deepseek-v41-flash-on-2x-dgx-spark-x86-offload-6adab604a517b426ded3916e)

<!-- pli:blurb:start -->

> Published by [PLI Labs](https://proprietarylegal.ai/labs.html), the open research arm of [Proprietary Legal Intelligence](https://proprietarylegal.ai). PLI builds AI systems around a legal organization's own work product, combining retrieval, staged workflows, validation, and attorney review. Visit [proprietarylegal.ai](https://proprietarylegal.ai) to explore our work and discuss what your organization needs.

<!-- pli:blurb:end -->

## Status

2026-10-23 — production: remote Engram has served since 2026-10-22; row exactness and long-context prefill are qualified on the tested image, with no demonstrated decode gain.

The standalone source copy needs a site-specific manifest and staging layout before deployment. It contains no weights or corpus.

## What it is

Engram retrieves token-history-dependent embedding rows and feeds them into the model's residual stream. DeepSeek's upstream description is linked under **Upstream claims**. This model has Engram at layers 1 and 14. Its rows contain 256 bytes of FP8 values and 8 bytes of scales, with 24 hash columns split across the tensor-parallel ranks.

The server loads and hashes the tables once into x86 RAM. Requests identify the table and row range. The engine overlay fetches rows through the included transport into registered staging buffers, preserving the image's stream handoff and lookup kernel. A timeout, transport error or refused identity returns the lookup to the image's local reader. Reconnection happens in the background.

## Why

Random disk reads can delay long-prompt prefill. Moving those reads to RAM removes much of that lookup latency. It frees the Sparks from the corresponding NVMe read work; it **does not free Spark GPU or unified memory**, because this baseline already stored the tables on disk. The local shards remain necessary for fallback.

## When to use it

Use it for long prompts when an x86 host has spare RAM and a direct RoCE link to each Spark. Measure cold and warm prompts separately. It is most useful when Engram reads limit prefill.

## When not to use it

Short prompts, already cached prefixes, a RAM-constrained host, or a topology without dedicated RoCE links may offer little benefit. Do not use it to claim more GPU memory or faster decode.

## Requirements

- Two DGX Spark GB10 nodes in TP2 and the HeNryous image `ghcr.io/henryous/deepseek-v41-1017-dgx-spark:v5`. Other images are unqualified.
- About 190 GiB of host RAM for the tables, plus connection buffers and operating-system headroom. Measured resident table memory was 189 GiB.
- A RoCE-capable NIC on each end. The tested links used ConnectX-7 on Spark and ConnectX-5 Ex on x86, 40 Gb/s DACs, Ethernet MTU 9000 and active verbs MTU 4096.
- A C++ compiler, libibverbs headers/runtime, pthreads, and the image's PyTorch/CUDA build environment for the overlay extension.
- Matching local Engram shards and a trusted manifest with shard sizes and hashes. Preserve identity checking. The manifest is an external deployment prerequisite in this standalone copy.

## Measured effect

Our inference tests used the public [EXL3 2.75 bpw served model](https://huggingface.co/ProprietaryLegal/DeepSeek-V4.1-Flash-1017-EXL3-2.75bpw-LegalWeighted-DGX-Spark), two GB10 nodes in TP2, the same target weights and on-Spark drafter, maximum length 327,680, thinking high, temperature 1 and top_p 1. Only the Engram row source changed. Dates below are UTC. Values are mean ± sample standard deviation where reported.

The following tests ran on 2026-10-22 at concurrency 1. Cold rows use 3 repeats; warm rows state their smaller sample counts.

| Context / metric / repeats if warm | Local disk | Remote RAM / RDMA | Change |
|---|---:|---:|---:|
| Cold 32K; TTFT | 15.53 ± 0.12 s | 14.70 ± 0.07 s | −5.4 % |
| Cold 150K; TTFT | 73.62 ± 0.11 s | 69.44 ± 0.04 s | −5.7 % |
| Warm 32K; TTFT; 2 repeats | 0.53 s | 0.53 s | 0 |
| Warm 150K; TTFT; 1 repeat | 0.96 s | 0.91 s | −5 % |
| Cold 150K prefill + 1,500 output tokens; task time | 100.35 ± 0.55 s | 95.17 ± 1.61 s | −5.2 % |

On the same hardware and configuration, also on 2026-10-22:

| Context / concurrency / repeats | Throughput or decode finding |
|---|---|
| Cold 32K; concurrency 2; 2 repeats per configuration | Aggregate 40.6 → 42.3 tok/s |
| Cold 32K; concurrency 3; 2 repeats per configuration | Aggregate 47.2 → 47.8 tok/s |
| Cold short through 150K; concurrency 1; 3 repeats | Derived decode time per round changed +0.4 to +3.3 %, within repeat spread; no demonstrated gain or cost |

Those historical decode values were derived from token rate and accepted drafts. They are not direct per-step measurements. Aggregate throughput includes prefill and differs from steady decode throughput.

| Date / exactness or reliability test / hardware / sampling | Result |
|---|---|
| 2026-10-22, count reconciled 2026-10-23; both Spark ranks against local shards; both layers; TCP and RDMA; standalone lookup tests, no inference context or concurrency setting; one recorded sweep per rank | 839,040 lookup entries per rank; 0 bad rows; wrong identity and out-of-range requests refused |
| 2026-10-22; RTX 3090 module comparison; image disk vs remote vs overlay disabled; both layers and ranks; one test suite | BF16 bit-identical outputs; the unsupported SM86 FP8 lookup path was emulated in this module test |
| 2026-10-22; GB10 TP2; greedy, thinking off, fresh cache; short, 8K and 32K prompts; repeated baseline vs itself and baseline vs remote | Remote outputs stayed within the baseline's own nondeterminism floor; this is not a long-context token-identity claim |
| 2026-10-22; full GB10 TP2 remote-Engram comparison suite, contexts and repeats above, concurrent groups included | 0 fallbacks, 0 transport errors, 0 refusals, 0 reconnects; decode-size lookup mean 50–60 µs |

| Date / server-only measurement / conditions | Result |
|---|---|
| 2026-10-22; x86 RAM server, both ~101.5 GB shards loaded in parallel with O_DIRECT and concurrent hashing; one reported load; no inference context or request concurrency | Each shard loaded in 63–67 s; the prior card rounded startup to ~70 s |
| 2026-10-22–2026-10-23; GB10 TP2 + x86 remote Engram and one RTX 3090 drafter; short through 300K and concurrent capacity suite; one suite counter snapshot, not independent repeats | Rank 0: 37.9 M rows per table; mean ~205–210 µs; 0 fallbacks and 0 transport errors |

## Upstream claims

[DeepSeek's inference source](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017/blob/main/inference/model.py) describes Engram as n-gram hash lookup embeddings, gated against the residual stream. This is the model maker's architectural description, not our performance result.

The HeNryous image and sudoingX recipe provide the disk-offload baseline. We do not substitute their disk microbenchmarks for our end-to-end measurements.

## Deploy

Build from the standalone repository root:

```bash
make -C code/transport
make -C code/engram/server
```

Supply verified shards and the compatible manifest yourself. Start the server on explicitly configured direct-link addresses; the executable supports repeated `--listen` arguments:

```bash
code/engram/server/engram_server --data /data/engram \
  --manifest /data/runtime/manifest.json \
  --listen X86_LINK_A_IP --listen X86_LINK_B_IP
```

The overlay currently expects a deployment-shaped runtime root. Prepare `/data/runtime/offload/engram` and `/data/runtime/offload/transport` from `code/engram` and `code/transport`, and place the trusted manifest at `/data/runtime/manifest.json`. Mount that root read-only into each serving container at the same path. The existing staging scripts assume the original deployment layout; adapt their copy list before using them with this standalone tree.

With the serve stopped, prebuild the extension using the image's Python and CUDA environment. Set these container variables, then import `vllm.models.deepseek_v41.common.engram` and call its `_load_engram_remote_ext()` before model loading:

```text
PYTHONPATH=/data/runtime/offload/engram/spark/py
DSV41_ENGRAM_OVERLAY=/data/runtime/offload/engram/spark/engram.py
VLLM_ENGRAM_REMOTE_ROOT=/data/runtime
VLLM_ENGRAM_REMOTE=rdma://X86_LINK_A_IP:18610@SPARK_A_LINK_IP,rdma://X86_LINK_B_IP:18610@SPARK_B_LINK_IP
VLLM_ENGRAM_REMOTE_TIMEOUT_MS=100
VLLM_ENGRAM_REMOTE_REQUIRED=0
```

Use required mode for qualification so an unavailable remote backend cannot silently become a local-disk benchmark. Retain the image's disk-offload configuration and local shards. Verify image hashes, table identities and growing lookup counters on both ranks, with no fallbacks during the measured interval. The detailed [integration notes](code/engram/INTEGRATION.md) describe the original layout and should be read with the standalone-layout limitation above.

## Rollback

Remove `DSV41_ENGRAM_OVERLAY`, the overlay `PYTHONPATH`, and the `VLLM_ENGRAM_REMOTE*` settings; restart the serve. The image's own disk reader resumes. If the x86 server disappears while serving, failed lookups fall back to exact local rows until reconnect succeeds. This can add timeout latency; it is not an instantaneous failover.

## Limits and known issues

- Gains concern prompt reading. Decode changes are within repeat variation.
- A direct link shares a Spark PCIe uplink with tensor-parallel traffic; see the benchmark-tools entry for loaded-link measurements.
- The overlay is pinned to the tested image's reader semantics. Keep identity checks and exact local fallback intact when porting.
- The standalone copy lacks the deployment manifest and original directory tree expected by the staging and control scripts. The server build is standalone; deployment orchestration needs the runtime layout above.
- Independent review found and fixed a gather-pool lifetime race and control-script process targeting defects before the production server replacement. Use the reviewed source rather than an older binary.

## Repository layout

| Path | Purpose |
|---|---|
| [code/engram/server](code/engram/server) | Resident table server, build rules and service/control examples |
| [code/engram/spark](code/engram/spark) | Engine overlay, remote extension, import hook and staging scripts |
| [code/engram/engram_proto.h](code/engram/engram_proto.h), [protocol implementation](code/engram/engram_proto.cpp), [client](code/engram/engram_client.h) | Table identities, request layout and client lookup semantics |
| [code/engram/layout.py](code/engram/layout.py) | Compute per-rank row ranges from model configuration |
| [code/engram/tests](code/engram/tests) | Synthetic tables, exactness clients, failure checks and GPU comparisons |
| [code/transport](code/transport) | Included C/C++ transport, Python binding, probes and tests |

## Tests

The CPU transport integration suite needs a local RoCE interface even for its same-host run; it exercises TCP and RDMA. After building, choose an unused port and the local direct-link IP:

```bash
OFL_LIB="$PWD/code/transport/libofl.so" \
  python code/transport/test_ofl.py --ip LOCAL_DIRECT_IP --port 18601
python code/engram/layout.py /data/model/config.json --tp 2
```

The layout command emits JSON; it is a utility, not a test suite. The reviewed transport suite recorded 26/26 checks passing, including timeout, late-write prevention and reconnect. Earlier publication reports did not record a standalone Engram CPU count. GPU extension/module tests need weights, CUDA and the image's disk reader; a CPU pass alone does not qualify the GB10 handoff. No live hardware tests were run for this documentation update.

## Credits and licences

Our code is Apache-2.0. The overlay and copied layout helpers derive from the vLLM/HeNryous serving stack, Apache-2.0; the overlay ports DeepSeek's MIT-licensed Engram reference. The sudoingX recipe supplies deployment context. [NOTICE](NOTICE) identifies the components and their full texts in [LICENSES](LICENSES). Runtime libraries and model weights are external dependencies.

## Related

| Entry | GitHub | Hugging Face |
|---|---|---|
| Async host-flag handoff | [Source](https://github.com/ProprietaryLegal/dsv41-async-hostflag-handoff) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-async-hostflag-handoff) |
| Remote RTX 3090 drafter | [Source](https://github.com/ProprietaryLegal/dsv41-remote-dspark-drafter-3090) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-remote-dspark-drafter-3090) |
| NVLink drafter | [Source](https://github.com/ProprietaryLegal/dsv41-dspark-drafter-tp2-nvlink) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-dspark-drafter-tp2-nvlink) |
| ofl transport | [Source](https://github.com/ProprietaryLegal/ofl-rdma-transport) | [Card](https://huggingface.co/ProprietaryLegal/ofl-rdma-transport) |
| Native SM86 drafter | [Source](https://github.com/ProprietaryLegal/dsv41-native-1017-drafter-sm86) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-native-1017-drafter-sm86) |
| Benchmark tools | [Source](https://github.com/ProprietaryLegal/dsv41-longctx-bench-tools) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-longctx-bench-tools) |
| Exact acceptance ceiling | [Source](https://github.com/ProprietaryLegal/dsv41-exact-acceptance-ceiling) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-exact-acceptance-ceiling) |
| Full stack overview | [Source](https://github.com/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) | [Card](https://huggingface.co/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) |
| Offload findings | [Source](https://github.com/ProprietaryLegal/dsv41-spark-offload-findings) | [Card](https://huggingface.co/ProprietaryLegal/dsv41-spark-offload-findings) |

Upstream forks: [HeNryous lineage](https://github.com/ProprietaryLegal/deepseek-v41-1017-dgx-spark) · [sudoingX lineage](https://github.com/ProprietaryLegal/dgx-spark-deepseek-v41).

[Collection](https://huggingface.co/collections/ProprietaryLegal/deepseek-v41-flash-on-2x-dgx-spark-x86-offload-6adab604a517b426ded3916e) · [Hub](https://github.com/ProprietaryLegal/pli-labs/tree/main/research/dgx-spark-offload).

<!-- pli:footer:start -->

---

## About PLI Labs

“Your proprietary work product is your proprietary intelligence.” PLI Labs is the open research arm of Proprietary Legal Intelligence. We publish legal model prunes, serving recipes for owned hardware, and open-source tools so readers can inspect our methods and test our claims before hiring us.

Founded by a practicing trial lawyer, PLI builds AI systems around a legal organization's own work product: retrieval, staged workflows, validation, and attorney review. The approach grew from systems running on live matters in a litigation practice. Our consulting work supports deployment through approved frontier APIs, private cloud, or open-weight models on hardware you own. Explore the research or contact us about your organization's work.

[Website](https://proprietarylegal.ai) · [PLI Labs](https://proprietarylegal.ai/labs.html) · [Research](https://proprietarylegal.ai/research/index.html) · [GitHub](https://github.com/ProprietaryLegal) · [Hugging Face](https://huggingface.co/ProprietaryLegal) · [Contact](https://proprietarylegal.ai/contact.html)

Proprietary Legal Intelligence LLC is a technology consultancy, not a law firm, and does not provide legal services or legal advice.

<!-- pli:footer:end -->
