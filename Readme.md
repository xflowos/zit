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
- **Snapshots** – Every commit/package version is a ZFS snapshot (instant, zero-copy)
- **Clones** – Writable branches without duplicating data (Copy-on-Write)
- **Rollback** – Instant recovery from bad state  

**Without ZFS:** Core magic is lost for now. Portable backends (IPFS, S3, etc.) planned as alternative providers.

### 2. rsync and the MAVLink Ecosystem - Giants of Sync and Messaging
**Key tools we interoperate with:**
- **[rsync](https://rsync.samba.org/)** – Legendary delta-transfer algorithm and tool
- **[MAVLink Protocol](https://mavlink.io/)** – Lightweight hybrid pub/sub + point-to-point messaging standard
- **[MAVProxy](https://ardupilot.org/mavproxy/)** & **[pymavlink](https://github.com/ArduPilot/pymavlink)** – Battle-tested ground station, proxy, and Python tools

**Giants behind them:**
- **Andrew "Tridge" Tridgell (OAM)** – Creator/maintainer of rsync (with Paul Mackerras), Samba; creator and lead maintainer of MAVProxy; original author of pymavlink; ArduPilot lead developer (fixed-wing/VTOL)
- **Lorenz Meier** – Creator of the MAVLink protocol (2009), PX4, Pixhawk, QGroundControl

**Why ZIT stands directly on these shoulders:**
- zit-sync (planned v0.5.0) will use **full rsync compatibility** for efficient remote deltas
- Direct interop with **MAVProxy/pymavlink** tools to leverage MAVLink’s hybrid pub/sub for lightweight metadata broadcasts, package announcements, and P2P coordination over TailScale

Huge respect to Andrew "Tridge" Tridgell – true Aussie GNU/CCC-style hacker legend – and Lorenz Meier for these foundational, interoperable tools that have powered real-world systems for decades.

### 3. [Tailscale](https://tailscale.com) - The SDN Magic for P2P Distribution
**What it is:** Zero-config, secure mesh VPN built on WireGuard – making any devices talk directly as if on the same LAN, anywhere.  
**Founders/Hackers:** Avery Pennarun (CEO), David Crawshaw, David Carney – true networking wizards turning NAT hell into seamless connectivity.  
**Why ZIT (and xFlowOS) loves it:**
- zit-sync (v0.5.0+) uses Tailscale for **P2P distribution, team package caching, and secure remote sync** – nodes discover and connect directly, no port forwarding nightmares.
- Enables distributed xFlowOS fleets to feel like one big local network.
- **Critically:** Their **huge free tier** (3 users, 100 devices – perfect for personal/homelab/open-source projects) makes this accessible to all hackers without barriers.

Massive props to the Tailscale team for building this game-changer and keeping a generous free plan alive. xFlowOS SDN wouldn't be the same without standing on these shoulders. 🤘

(Note: For fully self-hosted control server, the excellent community OSS [Headscale](https://github.com/juanfont/headscale) is compatible and recommended for larger/privacy-focused setups.)

### 4. [Charm.sh](https://charm.sh) - Beautiful Terminal UIs
**What it is:** Suite of tools for delightful terminal interfaces (Bubble Tea, Lip Gloss, Huh, Log).  
**Why ZIT uses it:**
- **Bubble Tea** – Interactive TUIs (progress bars, spinners, selection lists)
- **Lip Gloss** – Styled output (colors, borders, layouts)
- **Huh** – Beautiful forms and prompts
- **Log** – Structured, pretty logging  

**Without Charm:** ZIT would look like it’s from 1995 and be 10× harder to build.

---
## What is ZIT?
**ZIT is the datastore provider for xFlowOS.**  
It manages:
- **Source code** – Project repositories (like Git, but line-granular)
- **Files**, **blobs**, **blocks** (images, ISOs, etc.)

Using:
- **ZFS snapshots** – Instant, zero-copy versioning
- **Line-granular content addressing** – Maximum deduplication
- **DataStore indexing** – Fast queries and metadata
- **Beautiful TUI** – Charm.sh for a delightful experience

---
## Current Status
**⚠️ Work in Progress – v0.3.0**  
ZIT currently implements:
- ✅ Basic VCS operations (init, add, status, diff, branch, ls)
- ✅ Line-granular content addressing
- ✅ SQLite storage backend
- ✅ Beautiful Charm.sh TUI
- ✅ Multi-platform builds (Linux, macOS, Windows)

**Not yet implemented:**
- ❌ ZFS snapshot integration (planned v0.4.0)
- ❌ Commit/tag support (planned v0.4.0)
- ❌ Remote sync (planned v0.5.0)
- ❌ Merge operations (planned v0.6.0)

Installation and usage instructions describe the planned architecture. PRs very welcome :)

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
1. **[OpenZFS](https://openzfs.org)** – The foundation (snapshots, clones, dedup, compression)
2. **Andrew "Tridge" Tridgell** & **Lorenz Meier** – rsync, MAVProxy, pymavlink, MAVLink protocol
3. **Tailscale Team** (Avery Pennarun, David Crawshaw, David Carney) – Secure mesh networking magic
4. **[Charm](https://charm.sh)** – Beautiful terminal UIs (Bubble Tea, Lip Gloss, Huh, Log)
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) – Fast, secure hashing

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
I'm just one hacker (actually a chef, not even a pro dev)! Priorities:
- [ ] Peer review
- [ ] Failure
- [ ] More Peer review
- [ ] Benchmarks
- [ ] Test suite
- [ ] Real-world production failure → recovery → stability testing  

(This is a single-hacker WIP. Storing data in xFlowOS is no joke – real incidents, metrics, and battle-testing required.)

**Want to learn more about xFlowOS?**  
https://xflowos.au

---
*Zetabyte Information Tracker – Because ZFS makes everything better.*
