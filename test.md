# ZIT - Zetabyte Information Tracker

**Version:** 0.2.0  
**Purpose:** Line-granular version control and package management for xFlowOS  
**License:** GPL-3.0 (Commercial licensing available)  
**Author:** Shaun Lloyd (0@xflowos.au)

---

## Standing on the Shoulders of Giants

ZIT is built on battle-tested tools. We don't reinvent wheels.

### 1. [OpenZFS](https://openzfs.org) - Storage Foundation

**What it is:** Production-grade filesystem with Copy-on-Write, snapshots, compression, and deduplication.

**Why ZIT uses it:**
- **Snapshots** - Every commit/package version is a ZFS snapshot (instant, zero-copy)
- **Clones** - Writable branches without duplicating data (Copy-on-Write)
- **Block-level dedup** - Identical storage blocks stored once
- **Compression** - 60-80% space savings on text (transparent)
- **Rollback** - Instant recovery from bad state

**ZIT stores:**
```bash
rpool/u/pkg/node@20.0.0      # Package version as ZFS snapshot
rpool/u/src/myproject@main   # Source code repository
```

**Without ZFS:** ZIT falls back to basic file operations (slower, no snapshots, no dedup).

---

### 2. [mise-en-place](https://mise.jdx.dev) - Package Registry

**What it is:** Polyglot tool version manager with community-maintained package definitions.

**Why ZIT uses it:**
- **Community packages** - Hundreds of tools already defined (node, python, go, rust, etc.)
- **Package registry** - URLs, versions, checksums maintained by community  
- **Zero maintenance** - Don't define packages yourself
- **Version metadata** - What exists, where to get it, how to verify

**mise provides the WHAT and WHERE:**
```bash
mise says: "node@20.0.0 exists at https://nodejs.org/dist/v20.0.0/..."
ZIT says: "Thanks, I'll store that in rpool/u/pkg/node@20.0.0"
```

**Without mise:** You'd manually define every package, every version, forever.

---

### 3. [Charm.sh](https://charm.sh) - Beautiful Terminal UIs

**What it is:** Suite of tools for delightful terminal interfaces (Bubble Tea, Lip Gloss, Huh, Log).

**Why ZIT uses it:**
- **Bubble Tea** - Interactive TUIs (progress bars, spinners, selection lists)
- **Lip Gloss** - Styled output (colors, borders, layouts)
- **Huh** - Beautiful forms and prompts
- **Log** - Structured, pretty logging

**ZIT uses Charm for:** Beautiful terminal output with progress bars, styled diffs, and interactive prompts.

**Without Charm:** ZIT would work but look like it's from 1995.

---

## What is ZIT?

**ZIT is the version control and package management tool for xFlowOS.**

It manages:
- **Source code** - Project repositories (like Git, but line-granular)
- **Packages** - Tools, runtimes, binaries (like mise, but ZFS-backed)

Using:
- **ZFS snapshots** - Every version instant and zero-copy
- **Line-granular content addressing** - Deduplicate at line + block level
- **SQLite indexing** - Fast queries and metadata
- **Beautiful TUI** - Charm.sh for delightful experience

---

## Current Status

**⚠️ Work in Progress - v0.2.0**

ZIT currently implements:
- ✅ Basic VCS operations (init, add, status, diff, branch, ls)
- ✅ Line-granular content addressing
- ✅ SQLite storage backend
- ✅ Beautiful Charm.sh TUI
- ✅ Multi-platform builds (Linux, macOS, Windows)

**Not yet implemented:**
- ❌ ZFS snapshot integration (planned v0.3.0)
- ❌ Commit/tag support (planned v0.3.0)
- ❌ mise package management integration (planned v0.4.0)
- ❌ Remote sync (planned v0.5.0)
- ❌ Merge operations (planned v0.6.0)

**Installation and usage instructions below describe the planned architecture.**  
Current v0.2.0 works as a local-only VCS with basic operations.

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

## FAQ

**Q: Is ZFS required?**  
A: For core benefits (snapshots, clones, dedup), yes. ZIT falls back to basic operations without ZFS, but you lose the magic.

**Q: Can I use this without mise?**  
A: Yes, but you lose access to the community package registry. You'd define packages manually.

**Q: Does this replace Git?**  
A: For xFlowOS development, yes. For open-source collaboration on GitHub, no (yet - Git interop coming in v0.6.0).

**Q: Why line-granular instead of file-based like Git?**  
A: For codebases with high code similarity (monorepos, config files), line-granular provides 5-10x better deduplication.

**Q: What about macOS?**  
A: Install OpenZFS for macOS. Works great. See: https://openzfsonosx.org

**Q: Is this production-ready?**  
A: v0.2.0 is stable for local use. Remote features coming. Use with backups.

**Q: What's the commercial licensing?**  
A: GPL-3.0 for open source. Dual commercial licensing available for enterprises. Contact 0@xflowos.au

---

## Architecture

### Storage Layout

```
ZIT Repository on ZFS:
rpool/u/src/myproject/
├── .zit/
│   ├── pkg.db           # SQLite (line hashes, file definitions, metadata)
│   └── config           # Local settings
└── [working files]

ZFS Snapshots:
rpool/u/src/myproject@main        # Current branch state
rpool/u/src/myproject@commit-abc  # Each commit
rpool/u/src/myproject/feature-x   # Cloned branch

Package Storage:
rpool/u/pkg/node@18.0.0           # Package version snapshot
rpool/u/pkg/node@19.0.0           # Delta from @18.0.0
rpool/u/pkg/node@20.0.0           # Delta from @19.0.0
```

### How Content is Stored

1. **Ingest** - Files split into lines, each BLAKE3 hashed
2. **Store** - Unique lines stored once in SQLite
3. **Track** - Files = ordered sequences of line hashes
4. **Snapshot** - Commit creates ZFS snapshot of entire repository
5. **Dedup** - Identical lines across all files share storage
6. **Compress** - ZFS transparent compression on dataset

---

## Credits

**Standing on the shoulders of:**

1. **[OpenZFS](https://openzfs.org)** - The foundation (snapshots, clones, dedup, compression)
2. **[mise](https://mise.jdx.dev)** - Package registry and community
3. **[Charm](https://charm.sh)** - Beautiful terminal UIs (Bubble Tea, Lip Gloss, Huh, Log)

Also:
- [SQLite](https://sqlite.org) - Metadata and indexing
- [goreleaser](https://goreleaser.com) - Release automation
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) - Fast, secure hashing

**ZIT exists because these tools are excellent.**

---

## License

**GPL-3.0**

Commercial licensing available for proprietary modifications and enterprise support.  
Contact: 0@xflowos.au

---

## Author

**Shaun Lloyd**  
Email: 0@xflowos.au  
Web: https://xflowos.au

Part of the **xFlowOS ecosystem**.

---

## Next Steps

**Want to contribute?**
- Priority: ZFS send/receive for remote sync
- Priority: Line-level merge algorithm
- Priority: mise native integration
- See: https://github.com/xflowos/zit/issues

**Want commercial licensing?**
- Email: 0@xflowos.au

**Want to learn more about xFlowOS?**
- https://xflowos.au

---

**ZIT: Line-granular version control that respects your disk space.**

*Zetabyte Information Tracker - Because ZFS makes everything better.*
