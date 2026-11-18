# whisper-rs-sys 0.15.0, patched for talos-vox

This is the crates.io release of `whisper-rs-sys` 0.15.0, unmodified in the
first commit, plus fixes to the vendored whisper.cpp and ggml sources so
that running out of GPU memory returns an error instead of crashing the
process. talos-vox uses it through `[patch.crates-io]` at a pinned rev. See
talos-hq/talos-vox#4.

The fixes:

- `whisper_state::batch` is value-initialised. `whisper_init_state`'s error
  paths call `whisper_free_state`, which freed the uninitialised batch
  pointers (segfault when the KV cache doesn't fit). Already fixed upstream
  in ggml-org/whisper.cpp#4155.
- A model buffer that can't be allocated fails the load (`whisper_init_*`
  returns NULL) instead of being skipped and then aborting on
  `GGML_ASSERT(buffer)` while loading the weights. Same for the VAD model.
- A failed model load releases the buffers it already allocated
  (`whisper_free` instead of `delete ctx`), so a CPU retry doesn't run with
  that device memory still held.
- `whisper_kv_cache_init` frees its ggml context on failure, and
  `whisper_kv_cache_free` clears the buffer pointer.
- When `whisper_full_with_state` can't grow the KV cache for more decoders
  (beam search), it returns -7 without freeing the caller's state, which
  the caller would otherwise free a second time.
- ggml: `ggml_backend_sched_alloc_splits` checks the result of
  `ggml_gallocr_reserve_n`. When a compute buffer couldn't be allocated
  (e.g. `whisper_init_state` on a GPU with room for the KV cache but not
  the encoder's compute buffer), it went on to place tensors in the
  missing buffer and segfaulted. Backported from upstream ggml, which
  already checks it.

To move to a new whisper-rs-sys release: import the new `.crate` contents in
one commit, re-apply the later commits' changes (dropping anything upstream
has since fixed), and bump the rev in
talos-vox's `Cargo.toml`.
