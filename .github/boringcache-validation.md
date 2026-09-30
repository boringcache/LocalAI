# LocalAI llama.cpp compiler-cache validation

This fork compares registry-backed BuildKit layers with BoringCache's managed
layer cache plus persistence of the existing `/root/.ccache` mount. It uses
LocalAI's CPU amd64 matrix entry, unchanged `backend/Dockerfile.llama-cpp`,
compile script, compiler launchers, build parallelism, and 5 GB ccache limit.
The prebuilt `base-grpc-amd64` image is pinned by digest in `.boringcache.toml`.
The finished backend package is exported locally and checked on both arms.

The registry control uses a fork-owned GHCR cache because this fork has no
write credentials for LocalAI's Quay cache. It retains the upstream
`type=registry,mode=max` cache surface; this does not measure Quay transport
performance or inherit LocalAI's existing cache. Neither arm imports an
upstream layer cache or transports any other dependency cache. Cache publication
errors fail both arms. Each job uses a new standard GitHub-hosted amd64 runner
and a new builder. BoringCache's Action owns its managed builder lifecycle.

`.github/boringcache-upstream` selects the real upstream commit for both arms.
The source checkout contains no validation files. The initial sequence is:

| Phase | Upstream source | Purpose |
| --- | --- | --- |
| Cold | `73cb1c3fdf4795689f55e4b84e4e7bcfe4cf436f` | Seed both previously unused cache tags, four first-parent commits before the captured head |
| Warm | Same base | Measure unchanged-source layer reuse on fresh builders |
| Rolling 1 | `01db8f33c15cbf312eb39c964ea9b665573ebb23` | CrispASR backend version update |
| Rolling 2 | `7cadb5748a3312d31561f5cb9cb6d50c8e761371` | CED backend version update |
| Rolling 3 | `b93d111d9b7ac01c5ecde1020e2884baba788f0d` | NeMo backend version update |
| Rolling 4 | `2fa36e714789dc519ce64089f47c88ebffc18a51` | Parakeet feature with shared protobuf changes |

Each rolling phase advances exactly one first-parent upstream commit. The first
three change only an unrelated backend's Makefile. `COPY . /LocalAI` observes
each change; llama.cpp, its pinned dependency, compiler settings, Dockerfile,
and compile script are identical. The fourth also changes `backend/backend.proto`,
which is a real input to the llama.cpp gRPC build. The workflow does not force
cache invalidation or synthesize a source change. A layer hit can replay old
ccache output without compiling: assess native compiler hits only when the
compile RUN executes, and keep layer reuse separate.

`connect` enrolls the fork's GitHub Actions identity for the dedicated
`boringcache/localai-validation` workspace. Builds use OIDC rather than static
cache secrets. `cold` and `rolling` publish; `warm` is restore-only.
The pinned BoringCache Action selects its own released default CLI.

Retain raw job logs, source and workflow SHAs, runner and builder versions,
native ccache reports, package checksums, full build-step durations, product
cache evidence, and transfer/publication measurements where available.
Do not interpret an unavailable byte measurement as zero. Keep runner queue
time separate from build/job time. CPU results do not establish CUDA or ROCm
timings, full PR feedback latency, or willingness to adopt or pay.

Dispatch `.github/workflows/boringcache-validation.yml` on
`boringcache-validation` with `phase=connect`, then `cold` and `warm`.
For each rolling run, commit the selected source SHA to
`.github/boringcache-upstream`, push normally, and dispatch `phase=rolling`.
Keep both cache tags fixed throughout the sequence.
