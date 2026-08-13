# ONNX Runtime WebGPU Buffer Cache Repro

The ONNX Runtime WebGPU EP supports several storage-buffer cache modes in its
C++ core, but `onnxruntime-web` does not forward the option from JS, so web
apps are pinned to the default `bucket` mode. For static-shape models the
bucket rounding is expensive. This page makes the cost visible.

Live demo: <https://xiaofeihan1.github.io/onnxruntime-webgpu-buffer-cache-repro/>

Upstream issue: <https://github.com/microsoft/onnxruntime/issues/29016>

Upstream PR: <https://github.com/microsoft/onnxruntime/pull/29017>

## What the page does

It runs the same static-shape model twice per mode and compares:

- `Official bucket` — `onnxruntime-web@1.26.0` as published (default cache
  mode).
- `Patched simple` — the same `ort.webgpu.min.mjs` with one line added to
  forward `storageBufferCacheMode`, then `storageBufferCacheMode: 'simple'`.
- `Patched no cache` — the same patched bundle with
  `storageBufferCacheMode: 'lazyRelease'`. It does not reuse storage buffers,
  but delays destruction until pending WebGPU commands have been submitted.

The only difference between the patched modes is the value of the forwarded
option.

The literal `disabled` value is not usable with deferred command submission in
ONNX Runtime 1.26.0: it releases buffers immediately while commands that
reference them can still be waiting for submission. `lazyRelease` provides the
same no-cache behavior without destroying in-flight buffers.

Model: [Musetric RoFormer vocal separation](https://huggingface.co/musetric/vocal-separation-roformer-onnx)
(fp16 weights, fp32 I/O `stft_repr` / `masks: [1, 2050, 1101, 2]`, 741 MB).
It is downloaded from Hugging Face when a run starts and is not stored in this
repository. The input is a deterministic random tensor.

## What it measures

The page wraps `GPUDevice.createBuffer`, `GPUBuffer.destroy` and
`GPUQueue.submit`, then reports per mode:

- **Weights** — bytes still resident after session create (identical in both
  modes).
- **Peak live** — maximum bytes alive at any moment.
- **Run 1 / Run 2 allocations** — new buffer bytes created during each run.
  The model is static, so an ideal cache allocates nothing on run 2.
- **Latency** per run and **submit count**.

Per-mode tabs show the live-buffer histogram at the peak.

## License

MIT. The model is published separately by Musetric, also under MIT.
