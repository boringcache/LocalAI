# LocalAI llama.cpp CPU compiler-cache results

Six paired runs completed successfully on September 30, 2026: 12 builds on fresh standard GitHub-hosted Ubuntu 24.04 amd64 runners and fresh builders. The sequence started four first-parent commits before captured upstream head `2fa36e714789dc519ce64089f47c88ebffc18a51`, repeated the starting commit once, then advanced through four real commits. No source mutation or forced Docker invalidation was added.

The build uses upstream's unchanged Dockerfile, compile scripts, llama.cpp version, compiler settings, all CPU variants, gRPC/RPC targets, and 5 GB ccache limit. The control uses GHCR-backed BuildKit registry layers with `mode=max`; BoringCache uses managed BuildKit layers plus persistence of the existing `/root/.ccache` mount. GHCR substitutes for upstream Quay because this fork has no Quay write credentials. This comparison does not measure Quay transport performance.

| Phase / source | Registry build step | BoringCache build step | Registry compiler hits | BoringCache compiler hits | Run |
| --- | ---: | ---: | ---: | ---: | --- |
| Cold / `73cb1c3f` | 833 s | 768 s | 0/600 | 0/600 | [36727031006](https://github.com/boringcache/LocalAI/actions/runs/36727031006) |
| Same commit / `73cb1c3f` | 496 s | 91 s | 0/600 | 600/600 | [36729966767](https://github.com/boringcache/LocalAI/actions/runs/36729966767) |
| CrispASR Makefile / `01db8f33` | 644 s | 144 s | 0/600 | 600/600 | [36731435691](https://github.com/boringcache/LocalAI/actions/runs/36731435691) |
| CED Makefile / `7cadb574` | 835 s | 138 s | 0/600 | 600/600 | [36733307169](https://github.com/boringcache/LocalAI/actions/runs/36733307169) |
| NeMo Makefile / `b93d111d` | 855 s | 104 s | 0/600 | 600/600 | [36735571765](https://github.com/boringcache/LocalAI/actions/runs/36735571765) |
| Shared protobuf / `2fa36e71` | 879 s | 200 s | 0/600 | 594/600 | [36737750835](https://github.com/boringcache/LocalAI/actions/runs/36737750835) |

The compile RUN executed on both arms in every run. Upstream resets ccache statistics before compiling, so the hit counts above are live compiler measurements. The same-commit run rebuilt the compile layer; `.git` is included in the upstream local Docker context and varies between fresh checkouts, but the exact invalidating input was not isolated. The three unrelated Makefile changes are observed by upstream's broad `COPY . /LocalAI`. The fourth changes shared protobuf input and yielded six compiler misses on BoringCache. Individual misses were not traced to files.

Build-step times include local package export and publication when enabled. BoringCache also owns CLI/builder setup inside its synchronous Action. Registry builder setup occurs before its timed build step. Full job times, including checkout, disk preparation, verification, and cleanup, were:

| Phase | Registry full job | BoringCache full job |
| --- | ---: | ---: |
| Cold | 986 s | 937 s |
| Same commit | 682 s | 276 s |
| CrispASR | 851 s | 637 s |
| CED | 1,037 s | 290 s |
| NeMo | 992 s | 379 s |
| Shared protobuf | 1,061 s | 354 s |

The CrispASR BoringCache job spent 478 seconds in upstream disk preparation before its 144-second build step. Compiler reuse does not remove runner preparation or queueing costs. Hosted runner throughput varied, so timing differences also include runner and transfer variation.

The four recorded Dockerfile/compile-script/target-script/Makefile hashes are identical across all 12 jobs. Every provider pair produced identical package checksums. The final package differs from the starting package. The workflow verifies the exported backend files; model inference is outside this validation.

CLI telemetry was retained for the last three runs. It reports these compiler-cache transfers and successful mount publications:

| Run | ccache restore bytes transferred | ccache publish bytes transferred | Mount publication |
| --- | ---: | ---: | --- |
| CED | 33,046,247 | 33,100,514 | 1 completed, 0 errors |
| NeMo | 33,100,514 | 33,148,652 | 1 completed, 0 errors |
| Shared protobuf | 33,148,652 | 35,888,209 | 1 completed, 0 errors |

Mount restores took 3.021, 1.954, and 1.157 seconds respectively. These counters cover the compiler mount; complete network-transfer totals and equivalent registry counters are unavailable. All cold/forward cache exports succeeded with cache errors treated as failures. The same-commit run was restore-only.

The [validation plan](boringcache-validation.md), [workflow](workflows/boringcache-validation.yml), and [repository configuration](../.boringcache.toml) retain the settings, pins, and source sequence. Raw logs and evidence artifacts are attached to the linked runs. The Action and default CLI were v1.33.0. Builds used approved GitHub Actions OIDC access to the dedicated validation workspace.

These CPU amd64 results do not establish CUDA/ROCm speedups, historical GPU timings, complete PR feedback latency, or willingness to adopt or pay. Choose unused tags for both providers when repeating a cold comparison; the tags used here are now seeded.
