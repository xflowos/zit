# ZIT - Zetabyte Information Tracker

**Version:** 0.3.0  
**Purpose:** Line-granular version control and package management for xFlowOS  
**License:** GPL-3.0
**Author:** Shaun Lloyd (0@xflowos.au)

---

<img width="600" alt="Melt example" src="https://github.com/xflowos/zit/blob/main/zit-demo.gif">

---

## Standing on the Shoulders of Giants

ZIT is built on battle-tested tools. We don't reinvent wheels.

### 1. [OpenZFS](https://openzfs.org) - Storage Foundation

**What it is:** Production-grade filesystem with Copy-on-Write, snapshots, compression, and deduplication.

**Why ZIT uses it:**
- **Snapshots** - Every commit/package version is a ZFS snapshot (instant, zero-copy)
- **Clones** - Writable branches without duplicating data (Copy-on-Write)
- **Rollback** - Instant recovery from bad state

**Without ZFS:** nothing ;) "yet". portable ZIT is planned. IPFS, S3 etc. "just providers"

---

### 3. [Charm.sh](https://charm.sh) - Beautiful Terminal UIs

**What it is:** Suite of tools for delightful terminal interfaces (Bubble Tea, Lip Gloss, Huh, Log).

**Why ZIT uses it:**
- **Bubble Tea** - Interactive TUIs (progress bars, spinners, selection lists)
- **Lip Gloss** - Styled output (colors, borders, layouts)
- **Huh** - Beautiful forms and prompts
- **Log** - Structured, pretty logging

**ZIT uses Charm for:** Beautiful terminal output with progress bars, styled diffs, and interactive prompts.

**Without Charm:** ZIT would be 10x more difficult to create and look like it's from 1995.

---

## What is ZIT?

**ZIT is the datastore provider for xFlowOS.**

It manages:
- **Source code** - Project repositories (like Git, but line-granular)
- FILES
- BLOBS
- BLOCKS: img, iso, etc

Using:
- **ZFS snapshots** - Every version instant and zero-copy
- **Line-granular content addressing** - Deduplicate at line.
- **DataStore indexing** - Fast queries and metadata
- **Beautiful TUI** - Charm.sh for delightful experience

---

## Current Status

**⚠️ Work in Progress - v0.3.0**

ZIT currently implements:
- ✅ Basic VCS operations (init, add, status, diff, branch, ls)
- ✅ Line-granular content addressing
- ✅ SQLite storage backend
- ✅ Beautiful Charm.sh TUI
- ✅ Multi-platform builds (Linux, macOS, Windows)

**Not yet implemented:**
- ❌ ZFS snapshot integration (planned v0.4.0)
- ❌ Commit/tag support (planned v0.4.0)
- ❌ Remote sync (planned v0.5.0), rsync & rclone "acknowledgement Giant Legend Aussie Professor Andrew Tridgell https://en.wikipedia.org/wiki/Andrew_Tridgell
- ❌ Merge operations (planned v0.6.0)

**Installation and usage instructions below describe the planned architecture.**  
NOT YET :( PR welcome :)

---

## Roadmap

### v0.3.0 - Core VCS Features
- [x] Line-granular tracking
- [x] ZFS snapshot integration
- [x] Beautiful TUI (Charm.sh)
- [ ] Commit messages and metadata
- [ ] Branch merging (line-level)
- [ ] Tag support

### v0.4.0 - Package Management
- [ ] Native mise integration
- [ ] Automatic ZFS snapshots on install
- [ ] Package dedup statistics
- [ ] Version switching via clones

### v0.5.0 - Remote Sync
- [ ] ZFS send/receive over network
- [ ] TailScale P2P distribution
- [ ] Team package caching

### v0.6.0 - Advanced Features
- [ ] Line-level merge tool
- [ ] Interactive rebase
- [ ] Conflict resolution UI
- [ ] Git interop (import/export)

### v1.0.0 - Production Ready
- [ ] Full stability guarantees
- [ ] Complete documentation
- [ ] Performance benchmarks
- [ ] Migration tooling

---

## Credits

**Standing on the shoulders of:**

1. **[OpenZFS](https://openzfs.org)** - The foundation (snapshots, clones, dedup, compression)
3. **[Charm](https://charm.sh)** - Beautiful terminal UIs (Bubble Tea, Lip Gloss, Huh, Log)
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) - Fast, secure hashing

**ZIT exists because these tools are excellent.**

---

## Author

**Shaun Lloyd**  
Email: 0@xflowos.au  
Web: https://xflowos.au

Part of the **xFlowOS ecosystem**.

---

## Next Steps

**Want to contribute?**
- Priority: *. Im just one hacker, chef actually not even professional developer !
- [ ] Peer Review
- [ ] Benchmarks
- [ ] Test Suite
- [ ] Production -> Failure -> Recovery -> Stability.
      - This is a WIP, single hacker project. "Storing data - especially as xflowos is um techically "no fucking joke !"
      - Real world incidents, metrics and implementation is required.

**Want to learn more about xFlowOS?**
- https://xflowos.au

---

*Zetabyte Information Tracker - Because ZFS makes everything better.*
