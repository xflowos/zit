# ZIT - Zetabyte Information Tracker

---

Version: 0.3.0  
License: GPL-3.0
Author: Shaun Lloyd
Email: 0@xflowos.au  

---

<img width="600" alt="Melt example" src="https://github.com/xflowos/zit/blob/main/zit-demo.gif">

---


# About

ZFS filesystem with Copy-on-Write, snapshots, compression, and deduplication.


## Stack

| Giant | Website | Description |
| ----- | ------- | ----------- |
| OpenZFS | (https://openzfs.org) | The foundation (snapshots, clones, dedup, compression) |
| Charm | (https://charm.sh) | Beautiful terminal UIs (Bubble Tea, Lip Gloss, Huh, Log) |
| SQLite | (https://sqlite.org) | Metadata and indexing |
| BLAKE3 | (https://github.com/BLAKE3-team/BLAKE3) - Fast, secure hashing |


## Pipeline

| Stage | Description |
| ----- | ----------- |
| Ingest | Files split chunk, lines, of entire file, each BLAKE3 hashed. |
| Dedup | Identical Hash stored as reference |
| Store | Unique lines stored once in SQLite |
| Compress | ZFS transparent compression on dataset |
| Index | Files = ordered sequences of line hashes |
| Verify | Verify the zit store |
| Snapshot | Commit creates ZFS snapshot of entire repository |


---


## 1.[OpenZFS](https://openzfs.org) - Storage Foundation

| Feature | Description |
| ------- | ----------- |
| Snapshots | Every commit package version is a ZFS snapshot (instant, zero-copy) |
| Clones | Writable branches without duplicating data (Copy-on-Write) |
| Compression |  60-80% space savings on text (transparent) |
| Rollback | Instant recovery from bad state |


## 2.[Charm.sh](https://charm.sh) - Beautiful Terminal UIs

Charmed UI, and i could never have learned golang without the charmed maps. Love.

| Charm | Description |
| ----- | ----------- |
| Bubble Tea | Interactive TUIs (progress bars, spinners, selection lists) |
| Lip Gloss | Styled output (colors, borders, layouts) |
| Huh | Beautiful forms and prompts | 
| Log | Structured, pretty logging |



---


# Roadmap

0.3.1 - Core VCS Features
- [x] POC
- [x] Line-granular tracking
- [ ] Commit messages and metadata

0.4.x - ZFS
- [ ] ZFS intergration
- [ ] optimise and maximise zfs block with stream of compressed data.

0.5.x - Per file type handling, chunking
- [ ] Line-level merge tool
- [ ] Chuck size.
- [ ] 4kb ? ZFS 

0.6.x - GIT features
- [ ] Git interop (import/export)
- [ ] Branch merging (line-level)
- [ ] Tag support

1.0.0 - Stable
- [ ] Full stability guarantees
- [ ] Complete documentation
- [ ] Performance benchmarks
- [ ] Migration tooling


---

Part of the **xFlowOS**.
