# Local buffered read methods

**Use this when:** choosing how the standard local backend materializes a shard window into owned bytes.
**Source of truth:** `crates/ecstore/src/disk/local.rs` (`StdBackend::pread_bytes`, `select_local_read_copy_method`).

`RUSTFS_OBJECT_MMAP_READ_METHOD` is read once on first use; restart the process
when changing it. The setting applies after the caller selects materialized
local reading. It does not enable that path or override the streaming size cap,
batch shard reads, remote reads, an active io_uring backend, or successful
explicit O_DIRECT reads.

| Setting | Standard buffered backend behavior |
| --- | --- |
| Unset, on Linux | Use positioned `read_exact_at` for nonempty windows at most 256 KiB that are shorter than the opened file; use mmap then copy otherwise. |
| Unset, other Unix platforms | Use mmap then copy. |
| `mmap_copy` | Always use mmap then copy. |
| `direct_read_copy` | Always use buffered positioned `read_exact_at`. This does not mean O_DIRECT. |
| Empty, unknown or non-Unicode | Preserve the mmap fallback. |

The 256 KiB bound is the encoded shard window, including checksum bytes, not
the S3 response size. A full small file still uses mmap. Selection uses the
same opened descriptor's metadata or validated descriptor-cache snapshot;
existing bounds checks, exact-read errors, descriptor invalidation, checksum
verification and reclaim policy remain in force. Metrics report the method
actually selected. Explicit O_DIRECT failures still fall back through this
buffered selection.

The partial-window rule aims to reduce page-fault read amplification for short
ranges while retaining the full-file method. It is not a memory quota or a
universal performance guarantee. Compare complete PUT, full GET and Range
workloads before deployment. Set `mmap_copy` to retain the prior buffered
method for an operational comparison.
