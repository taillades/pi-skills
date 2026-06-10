---
name: atmo-bundle-checkpoint
description: Bundle Atmo checkpoints into dev/checkpoint, CPU-compiled, or GPU-compiled bundles and optionally push them to BundleHub. Use when asked to wrap, bundle, export, package, compile, or publish a checkpoint.
---

# Atmo Bundle Checkpoint

Bundle an Atmo model checkpoint safely.

## Required bundle type

If the user does not specify the bundle kind, ask:

```text
What kind of bundle do you want?
1. dev/checkpoint bundle: checkpoint + config, no compilation
2. compiled CPU bundle
3. compiled GPU bundle
```

Do not assume dev.

## Interpret explicit requests

| User request | Bundle kind |
|---|---|
| “dev bundle” | dev/checkpoint |
| “checkpoint bundle” | dev/checkpoint |
| “source bundle” | dev/checkpoint |
| “wrap checkpoint, no compile” | dev/checkpoint |
| “no need to compile” | dev/checkpoint |
| “compiled CPU bundle” | compiled CPU |
| “CPU bundle” | compiled CPU |
| “compiled GPU bundle” | compiled GPU |
| “GPU bundle” | compiled GPU |
| “compiled bundle” | ask CPU or GPU |
| “dynamic bundle” | ask CPU or GPU unless stated |
| “bundle this checkpoint” | ask bundle kind |

## Modes

- `dev`: copies checkpoint into bundle with config/metadata/manifest; no compilation.
- `compiled CPU`: runs `torch.export` for CPU runtime.
- `compiled GPU`: runs `torch.export` for GPU runtime.
- Avoid `prod` unless the user explicitly asks for pickled-model bundle.

## Naming norms

Bundle tags must identify artifact type and runtime.

Use:

```text
<run-or-model>-epoch-<epoch>-dev
<run-or-model>-epoch-<epoch>-compiled-cpu
<run-or-model>-epoch-<epoch>-compiled-gpu
<run-or-model>-epoch-<epoch>-dynamic-cpu
<run-or-model>-epoch-<epoch>-dynamic-gpu
```

Examples:

```text
6eg192jb-epoch-13-dev
6eg192jb-epoch-13-compiled-cpu
6eg192jb-epoch-13-compiled-gpu
6eg192jb-epoch-13-dynamic-gpu
```

BundleHub URI:

```text
bundlehub://<namespace>/<bundle-name>:<tag>
```

Example:

```text
bundlehub://hanscom/ar:6eg192jb-epoch-13-dev
```

Rules:

- Never reuse a tag for different artifact types.
- Never overwrite existing BundleHub tags without explicit confirmation.
- Avoid ambiguous tags like `latest`, `test`, or bare run IDs unless user explicitly asks.
- For checkpoint-only bundles, use `-dev`.
- For compiled bundles, include `-cpu` or `-gpu`.
- If bundle has dynamic-shape export support, include `dynamic`.
- If unsure whether compiled output is dynamic, use `compiled`, not `dynamic`.

## Inputs to collect

Ask for missing required inputs:

1. checkpoint path or S3 URI
2. bundle kind:
   - dev/checkpoint
   - compiled CPU
   - compiled GPU
3. config path, if not adjacent to checkpoint
4. bundle namespace/name, e.g. `hanscom/ar`
5. desired tag, or permission to derive one from checkpoint path
6. whether to push to BundleHub

## Paths

Use local paths:

```bash
/home/yves/data/checkpoints/<checkpoint-parent>/<checkpoint-file>
/home/yves/data/bundles/<bundle-tag>
```

For S3 checkpoints:

```bash
mkdir -p /home/yves/data/checkpoints/<checkpoint-parent>
aws s3 cp <s3-uri> /home/yves/data/checkpoints/<checkpoint-parent>/<checkpoint-file>
```

## Build commands

### Dev/checkpoint bundle

```bash
bazel run --config=agent //atmo/atmonet/regional_forecast:export_autoregressive_bundle -- \
  <checkpoint-path> \
  <bundle-dir> \
  --mode dev
```

With explicit config:

```bash
bazel run --config=agent //atmo/atmonet/regional_forecast:export_autoregressive_bundle -- \
  <checkpoint-path> \
  <bundle-dir> \
  --config-path <config-path> \
  --mode dev
```

### CPU compiled bundle

```bash
bazel run --config=agent --@//runtimes:gpu=false \
  //atmo/atmonet/regional_forecast:export_autoregressive_bundle -- \
  <checkpoint-path> \
  <bundle-dir> \
  --mode compiled
```

With explicit config:

```bash
bazel run --config=agent --@//runtimes:gpu=false \
  //atmo/atmonet/regional_forecast:export_autoregressive_bundle -- \
  <checkpoint-path> \
  <bundle-dir> \
  --config-path <config-path> \
  --mode compiled
```

### GPU compiled bundle

```bash
bazel run --config=agent --@//runtimes:gpu=true \
  --run_under=/home/yves/atmo/local/nvidia_env.sh \
  //atmo/atmonet/regional_forecast:export_autoregressive_bundle -- \
  <checkpoint-path> \
  <bundle-dir> \
  --mode compiled
```

With explicit config:

```bash
bazel run --config=agent --@//runtimes:gpu=true \
  --run_under=/home/yves/atmo/local/nvidia_env.sh \
  //atmo/atmonet/regional_forecast:export_autoregressive_bundle -- \
  <checkpoint-path> \
  <bundle-dir> \
  --config-path <config-path> \
  --mode compiled
```

## Verify local bundle

Always inspect before pushing:

```bash
find <bundle-dir> -maxdepth 2 -type f | sort
cat <bundle-dir>/manifest.txt
```

Expected dev bundle:
- checkpoint file
- `config.yaml`
- `metadata.json`
- `manifest.txt`
- no compiled model artifact

Expected compiled bundle:
- compiled model artifact(s)
- `config.yaml`
- `metadata.json`
- `manifest.txt`

If expected files are missing, stop.

## Push to BundleHub

Push only after verification.

```bash
bazel run --config=agent //atmo/infra/bundlehub:cli -- \
  push <bundle-dir> bundlehub://<namespace>/<bundle-name>:<tag>
```

Example:

```bash
bazel run --config=agent //atmo/infra/bundlehub:cli -- \
  push /home/yves/data/bundles/6eg192jb-epoch-13-dev \
  bundlehub://hanscom/ar:6eg192jb-epoch-13-dev
```

## Verify BundleHub

After push, verify by resolving or pulling the bundle if supported:

```bash
bazel run --config=agent //atmo/infra/bundlehub:cli -- \
  pull bundlehub://<namespace>/<bundle-name>:<tag>
```

Then inspect cached bundle files if available.

## Safety checks

- If bundle kind is missing, ask before doing anything.
- If output bundle dir exists, ask before deleting/reusing.
- If BundleHub tag might already exist, ask before overwriting.
- If checkpoint config is missing, ask for config path.
- If user says “compiled” but not CPU/GPU, ask.
- If user says “CPU/GPU” and “no compile”, clarify because CPU/GPU only matters for compiled bundles.

## Response format

Return:

```text
Built bundle:
- kind: dev/checkpoint | compiled CPU | compiled GPU
- checkpoint: ...
- config: ...
- local bundle: ...
- BundleHub URI: ...

Validated:
- manifest: ...
- artifact type: checkpoint | compiled
```

If not pushed:

```text
Bundle built locally but not pushed.
```

## Example: explicit no-compile request

User gives:

```text
s3://atmo-hanscom/checkpoints/6eg192jb-epoch-13/autoregressive_fourier-013-0.1206.ckpt
```

and says:

```text
No need to compile it.
```

This means dev/checkpoint bundle:

```bash
aws s3 cp \
  s3://atmo-hanscom/checkpoints/6eg192jb-epoch-13/autoregressive_fourier-013-0.1206.ckpt \
  /home/yves/data/checkpoints/6eg192jb-epoch-13/autoregressive_fourier-013-0.1206.ckpt

bazel run --config=agent //atmo/atmonet/regional_forecast:export_autoregressive_bundle -- \
  /home/yves/data/checkpoints/6eg192jb-epoch-13/autoregressive_fourier-013-0.1206.ckpt \
  /home/yves/data/bundles/6eg192jb-epoch-13-dev \
  --mode dev

find /home/yves/data/bundles/6eg192jb-epoch-13-dev -maxdepth 2 -type f | sort
cat /home/yves/data/bundles/6eg192jb-epoch-13-dev/manifest.txt

bazel run --config=agent //atmo/infra/bundlehub:cli -- \
  push /home/yves/data/bundles/6eg192jb-epoch-13-dev \
  bundlehub://hanscom/ar:6eg192jb-epoch-13-dev
```
