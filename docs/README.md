# ODFileSystem — Architecture & Engineering Documentation

> A read-only optical-disc filesystem for the Amiga, built as a clean AmigaDOS
> handler over a portable, host-testable core with pluggable format backends.

This directory is the engineering guide to ODFileSystem (ODFS). It is written for
contributors who need to *change* the code, not just use it — so it favours the
"why" behind the design, the invariants you must not break, and the sharp edges
that have already drawn blood.

If you read nothing else, read **[overview.md](overview.md)** first. It frames the
whole system in one page and tells you which deeper document answers your
question.

## Reading guide

| Document | What it covers | Read it when… |
| --- | --- | --- |
| [overview.md](overview.md) | The 10,000-foot view: problem, layers, the central data flow, the cast of characters | Always start here |
| [architecture.md](architecture.md) | The five-layer architecture, the object model, the backend contract, format precedence | You need the structural map |
| [core-layer.md](core-layer.md) | `odfs_mount`, the node model, the LRU block cache, charset, name dedup, ancestry, logging, the allocator seam | You're touching `core/` |
| [media-layer.md](media-layer.md) | The `odfs_media_t` vtable, the host image/`.cue` backend, multisession discovery, the TOC model | You're touching device/image I/O or sessions |
| [backends-iso-family.md](backends-iso-family.md) | ISO 9660, Rock Ridge (SUSP/RRIP + Amiga `AS`), Joliet | You're touching the ISO-family parsers |
| [backends-udf-hfsplus.md](backends-udf-hfsplus.md) | UDF, HFS, HFS+ — descriptor chains and B-tree traversal | You're touching UDF or HFS/HFS+ |
| [backend-cdda.md](backend-cdda.md) | Audio CDs as virtual WAV/AIFF, CDDB/CD-Text synthesis, the mixed-mode graft | You're touching audio support |
| [amiga-handler.md](amiga-handler.md) | The AmigaDOS packet handler: lifecycle, locks, file handles, every packet, Workbench integration | You're touching `platform/amiga/` |
| [data-flows.md](data-flows.md) | End-to-end sequence diagrams: insert → mount, `Lock`, `Examine`/`ExNext`, `Read`, eject | You want to trace a request across all layers |
| [build-system.md](build-system.md) | Targets, the m68k cross toolchain, feature flags, size budgets, ADF images | You're touching the `Makefile` or CI |
| [rom-profile.md](rom-profile.md) | The minimal ROM-capable subset and its size budget | You're working on the ROM build |
| [testing.md](testing.md) | Unit, golden, malformed, fuzz, and integration testing strategy | You're adding or running tests |

## The one-paragraph summary

A disc is inserted. The AmigaDOS handler (`platform/amiga/handler_main.c`) wakes,
reads the drive's table of contents, and asks the portable core
(`core/mount.c`) to mount it. The core walks a precedence-ordered table of
**format backends** (`backends/*`), each of which can *probe* the media and, if it
recognises the on-disc structures, *mount* it and hand back a root
`odfs_node_t`. Every byte the backends read passes through an LRU **block cache**
(`core/cache_block.c`) sitting on a thin **media vtable** (`include/odfs/media.h`)
that abstracts "a real SCSI/IDE drive" from "an image file on your Mac". From then
on, AmigaDOS packets — `Lock`, `Examine`, `Read` — are translated by the handler
into core calls (`odfs_lookup`, `odfs_readdir`, `odfs_read`), and the core
dispatches them to whichever backend owns the node. The same core, the same
backends, and the same tests run unchanged on a host machine against image files,
which is why most of this filesystem can be developed without an Amiga in the room.

## A note on accuracy

Every non-obvious claim in these documents is anchored to a `file:line`
reference at the time of writing. Line numbers drift; treat them as a starting
coordinate, not gospel, and grep for the named symbol if the line has moved.
Where a document describes a *limitation* (single-extent reads, data-fork-only
HFS, O(n²) `ExNext`), that is a deliberate, documented trade-off — not a bug to be
"fixed" without understanding why it is the way it is.
